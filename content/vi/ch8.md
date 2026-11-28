---
translationKey: ddia-v2-ch8
title: Giao dịch
book_kind: chapter
book_number: "8"
book_part: II
weight: 208
math: true
breadcrumbs: false
---

<a id="ch_transactions"></a>

![](/map/ch07.png)

> *Một số tác giả cho rằng two-phase commit tổng quát quá đắt, gây ra các vấn đề về hiệu năng hoặc tính sẵn sàng. Chúng tôi cho rằng tốt hơn là để lập trình viên ứng dụng xử lý các vấn đề hiệu năng do giao dịch bị lạm dụng và trở thành nút thắt cổ chai, thay vì luôn bắt họ phải tìm cách lách vì thiếu giao dịch.*
>
> James Corbett và cộng sự, *Spanner: Google's Globally-Distributed Database* (2012)

Trong thực tế khắc nghiệt của các hệ thống dữ liệu, nhiều thứ có thể trục trặc:

* Phần mềm hoặc phần cứng của cơ sở dữ liệu có thể hỏng bất cứ lúc nào (kể cả giữa chừng một thao tác ghi).
* Ứng dụng có thể sập bất cứ lúc nào (kể cả giữa một chuỗi thao tác).
* Mạng bị gián đoạn có thể bất ngờ cắt đứt kết nối giữa ứng dụng và cơ sở dữ liệu, hoặc giữa các node cơ sở dữ liệu với nhau.
* Nhiều client có thể ghi vào cơ sở dữ liệu cùng lúc và ghi đè thay đổi của nhau.
* Client có thể đọc phải dữ liệu vô nghĩa vì dữ liệu mới chỉ được cập nhật một phần.
* Race condition giữa các client có thể gây ra những lỗi gây bất ngờ.

Để đáng tin cậy, hệ thống phải xử lý được các sự cố này và bảo đảm chúng không làm cả hệ thống hỏng một cách thảm họa. Tuy nhiên, hiện thực hóa cơ chế chịu lỗi đòi hỏi rất nhiều công sức: phải cân nhắc kỹ mọi thứ có thể sai, và phải thử nghiệm kỹ lưỡng để biết chắc giải pháp thật sự hiệu quả.

Suốt nhiều thập kỷ, *giao dịch* (*transaction*) là cơ chế được chọn để đơn giản hóa các vấn đề này. Ứng dụng dùng giao dịch để gộp nhiều thao tác đọc và ghi thành một đơn vị logic. Về mặt khái niệm, mọi thao tác đọc và ghi trong giao dịch được thực hiện như một thao tác duy nhất: hoặc cả giao dịch thành công (*commit*), hoặc thất bại (*abort*, *rollback*). Nếu thất bại, ứng dụng có thể thử lại an toàn. Nhờ giao dịch, việc xử lý lỗi của ứng dụng đơn giản hơn nhiều, vì ứng dụng không phải lo về lỗi từng phần — tức trường hợp một số thao tác thành công còn số khác thất bại vì bất cứ lý do gì.

Nếu đã làm việc với giao dịch nhiều năm, bạn có thể cho rằng chúng hiển nhiên phải có, nhưng giao dịch không phải quy luật tự nhiên. Chúng được tạo ra với một mục đích: *đơn giản hóa* *mô hình lập trình* (*programming model*) cho các ứng dụng truy cập cơ sở dữ liệu. Với giao dịch, ứng dụng có thể bỏ qua một số kịch bản lỗi tiềm ẩn và vấn đề đồng thời, vì cơ sở dữ liệu lo thay (ta gọi đó là các *bảo đảm an toàn*, *safety guarantee*).

Không phải ứng dụng nào cũng cần giao dịch. Đôi khi nên nới lỏng các bảo đảm của giao dịch, hay thậm chí bỏ hẳn (chẳng hạn để tăng hiệu năng hoặc tính sẵn sàng). Một số thuộc tính an toàn cũng đạt được mà không cần giao dịch. Mặt khác, giao dịch giúp tránh được vô số rắc rối: chẳng hạn, nguyên nhân kỹ thuật đằng sau vụ bê bối Horizon của Bưu điện Anh (xem ["Độ tin cậy quan trọng đến mức nào?"](/ch2#sidebar_reliability_importance)) rất có thể là hệ thống kế toán bên dưới thiếu giao dịch ACID[^1].

Làm sao biết bạn có cần giao dịch hay không? Để trả lời, trước hết ta phải hiểu chính xác giao dịch có thể đem lại những bảo đảm an toàn nào, và cái giá phải trả. Thoạt nhìn giao dịch có vẻ đơn giản, nhưng thực ra có rất nhiều chi tiết tinh tế mà quan trọng.

Chương này sẽ xem xét nhiều trường hợp có thể xảy ra sự cố, và tìm hiểu các thuật toán cơ sở dữ liệu dùng để phòng tránh. Ta sẽ đặc biệt đi sâu vào lĩnh vực *kiểm soát đồng thời* (*concurrency control*), bàn về các loại race condition có thể xảy ra và cách cơ sở dữ liệu hiện thực hóa các mức cô lập như read committed, snapshot isolation và serializable.

Kiểm soát đồng thời quan trọng với cả cơ sở dữ liệu đơn node lẫn phân tán. Ở phần sau của chương, trong mục ["Giao dịch phân tán"](/ch8#sec_transactions_distributed), ta sẽ xem xét giao thức *commit hai pha* (*two-phase commit*, 2PC) và các thách thức của việc đạt tính nguyên tử trong giao dịch phân tán.

## Giao dịch rốt cuộc là gì? {#sec_transactions_overview}

Ngày nay, hầu như mọi cơ sở dữ liệu quan hệ và một số cơ sở dữ liệu phi quan hệ đều hỗ trợ giao dịch. Phần lớn đi theo phong cách do IBM System R, cơ sở dữ liệu SQL đầu tiên, mở đường vào năm 1975[^2] [^3] [^4]. Dù một số chi tiết hiện thực đã thay đổi, ý tưởng chung gần như không đổi suốt 50 năm: hỗ trợ giao dịch trong MySQL, PostgreSQL, Oracle, SQL Server… giống System R một cách lạ lùng.

Cuối thập niên 2000, các cơ sở dữ liệu phi quan hệ (NoSQL) bắt đầu nổi lên. Chúng muốn phá vỡ cục diện hiện hữu của cơ sở dữ liệu quan hệ bằng cách đưa ra các lựa chọn mô hình dữ liệu mới (xem [Chương 3](/ch3#ch_datamodels)) và tích hợp sẵn sao chép (*replication*) ([Chương 6](/ch6#ch_replication)) cùng sharding ([Chương 7](/ch7#ch_sharding)). Giao dịch là nạn nhân chính của làn sóng này: nhiều cơ sở dữ liệu thế hệ mới bỏ hẳn giao dịch, hoặc định nghĩa lại từ "giao dịch" để chỉ một bộ bảo đảm yếu hơn nhiều so với trước đây.

Sự thổi phồng quanh các cơ sở dữ liệu phân tán NoSQL tạo ra một quan niệm phổ biến rằng giao dịch về bản chất không thể mở rộng, và hệ thống quy mô lớn nào muốn hiệu năng tốt, tính sẵn sàng cao đều phải bỏ giao dịch. Gần đây, quan niệm đó đã bị chứng minh là sai. Các cơ sở dữ liệu "NewSQL" như CockroachDB[^5], TiDB[^6], Spanner[^7], FoundationDB[^8] và YugabyteDB cho thấy hệ thống giao dịch vẫn mở rộng được tới lượng dữ liệu lớn và thông lượng cao. Các hệ thống này kết hợp sharding với giao thức đồng thuận ([Chương 10](/ch10#ch_consistency)) để cung cấp bảo đảm ACID mạnh ở quy mô lớn.

Tuy vậy, không có nghĩa hệ thống nào cũng phải hỗ trợ giao dịch. Như mọi lựa chọn thiết kế kỹ thuật khác, giao dịch có ưu điểm và hạn chế. Để hiểu sự đánh đổi, ta cần đi vào chi tiết các bảo đảm mà giao dịch có thể đem lại — cả khi hoạt động bình thường lẫn trong những tình huống cực đoan nhưng hoàn toàn có thể xảy ra.

### Ý nghĩa của ACID {#sec_transactions_acid}

Các bảo đảm an toàn của giao dịch thường được mô tả bằng từ viết tắt quen thuộc *ACID*, viết tắt của *tính nguyên tử* (*Atomicity*), tính *nhất quán* (*consistency*), tính *cô lập* (*isolation*) và *tính bền vững* (*Durability*). Thuật ngữ này do Theo Härder và Andreas Reuter đặt ra năm 1983[^9], nhằm xây dựng một cách diễn đạt chính xác về cơ chế chịu lỗi trong cơ sở dữ liệu.

Tuy nhiên, trên thực tế, mỗi cơ sở dữ liệu hiện thực hóa ACID một khác. Chẳng hạn, như ta sẽ sớm thấy, ý nghĩa của *tính cô lập* còn khá nhiều mơ hồ[^10]. Ý tưởng chung thì hợp lý, nhưng "quỷ nằm ở chi tiết". Ngày nay, khi một hệ thống tự nhận là "tuân thủ ACID", rất khó biết thực sự nó bảo đảm điều gì. Đáng tiếc, ACID gần như đã trở thành thuật ngữ tiếp thị.

(Các hệ thống không tuân thủ ACID đôi khi được gọi là *BASE*, viết tắt của *Basically Available* (về cơ bản là sẵn sàng), *Soft state* (trạng thái mềm) và *Eventual consistency* (nhất quán cuối cùng)[^11]. Định nghĩa này còn mơ hồ hơn cả ACID. Định nghĩa duy nhất hợp lý của BASE dường như là "không ACID". Nói cách khác, nó có thể mang gần như bất kỳ nghĩa nào bạn muốn.)

Hãy đào sâu định nghĩa của tính nguyên tử, tính nhất quán, tính cô lập và tính bền vững, để làm rõ hơn ý nghĩa của giao dịch.

#### Tính nguyên tử {#sec_transactions_acid_atomicity}

Nói chung, *nguyên tử* (*atomic*) là thứ không thể chia nhỏ hơn. Từ này mang nghĩa tương tự nhưng hơi khác nhau ở những mảng khác nhau của tin học. Ví dụ, trong lập trình đa luồng, nếu một luồng thực hiện một thao tác nguyên tử thì luồng khác không thể thấy thao tác đó mới hoàn thành một nửa. Hệ thống chỉ ở trạng thái trước hoặc sau thao tác, không có trạng thái lưng chừng.

Ngược lại, tính nguyên tử trong ACID *không* liên quan đến đồng thời. Nó không mô tả điều gì xảy ra khi nhiều tiến trình cùng truy cập một dữ liệu, đó là việc của *tính cô lập*, chữ *I* trong ACID (xem ["Tính cô lập"](/ch8#sec_transactions_acid_isolation)).

Tính nguyên tử của ACID mô tả một tình huống khác: client muốn thực hiện nhiều thao tác ghi, nhưng sau khi xử lý xong một số thì gặp sự cố — ví dụ tiến trình sập, kết nối mạng đứt, đĩa đầy, hoặc vi phạm một ràng buộc toàn vẹn nào đó. Nếu các thao tác ghi này được gộp vào một giao dịch nguyên tử mà sự cố khiến giao dịch không thể hoàn tất (*commit*), thì giao dịch bị *abort* và cơ sở dữ liệu phải loại bỏ hoặc hoàn tác mọi thao tác ghi mà giao dịch đã thực hiện.

Không có tính nguyên tử, nếu có lỗi giữa chừng khi áp dụng nhiều thay đổi, ứng dụng khó biết thay đổi nào đã có hiệu lực và thay đổi nào chưa. Ứng dụng có thể thử lại, nhưng như vậy có nguy cơ áp dụng cùng một thay đổi hai lần, gây ra dữ liệu trùng lặp hoặc sai. Tính nguyên tử đơn giản hóa vấn đề này: nếu giao dịch đã bị abort, ứng dụng biết chắc nó không thay đổi gì, nên có thể thử lại an toàn.

Khả năng abort giao dịch khi có lỗi và loại bỏ mọi thao tác ghi của nó chính là đặc điểm xác định của tính nguyên tử trong ACID. Có lẽ gọi là *khả năng hủy bỏ* (*abortability*) sẽ chính xác hơn *tính nguyên tử*, nhưng vì "tính nguyên tử" đã thành cách gọi quen thuộc nên cuốn sách vẫn dùng thuật ngữ này.

#### Tính nhất quán {#sec_transactions_acid_consistency}

Từ *nhất quán* (*consistency*) bị lạm dụng nặng nề:

* Ở [Chương 6](/ch6#ch_replication), ta đã bàn về *nhất quán giữa các bản sao* và vấn đề *nhất quán cuối cùng* (*eventual consistency*) trong các hệ thống sao chép bất đồng bộ (*asynchronous replication*) (xem ["Vấn đề của replication lag"](/ch6#sec_replication_lag)).
* *Snapshot nhất quán* (*consistent snapshot*) của cơ sở dữ liệu, chẳng hạn dùng để backup, là ảnh chụp toàn bộ cơ sở dữ liệu tại một thời điểm. Chính xác hơn, nó nhất quán với quan hệ happens-before (xem ["Quan hệ “xảy ra trước” và tính đồng thời"](/ch6#sec_replication_happens_before)): nếu snapshot chứa giá trị được ghi tại một thời điểm nào đó, thì nó cũng phản ánh mọi thao tác ghi xảy ra trước thời điểm đó.
* *Consistent hashing* là một cách chia shard mà một số hệ thống dùng để cân bằng lại (rebalance) (xem ["Consistent hashing"](/ch7#sec_sharding_consistent_hashing)).
* Trong định lý CAP (xem [Chương 10](/ch10#ch_consistency)), từ *nhất quán* chỉ *linearizability* (xem ["Linearizability"](/ch10#sec_consistency_linearizability)).
* Trong ngữ cảnh ACID, *tính nhất quán* chỉ "trạng thái tốt" của cơ sở dữ liệu theo định nghĩa riêng của từng ứng dụng.

Thật không may, cùng một từ mà có ít nhất năm nghĩa khác nhau.

Ý tưởng cơ bản của tính nhất quán trong ACID là: một số khẳng định về dữ liệu (tức các *bất biến*, *invariant*) phải luôn đúng. Ví dụ, trong hệ thống kế toán, tổng ghi có và ghi nợ trên mọi tài khoản phải luôn cân bằng. Nếu giao dịch bắt đầu với cơ sở dữ liệu thỏa các bất biến và mọi thao tác ghi trong giao dịch giữ được tính hợp lệ đó, bạn có thể chắc chắn các bất biến luôn được thỏa. (Bất biến có thể tạm thời bị phá vỡ trong lúc giao dịch chạy, nhưng đến khi commit thì phải được thỏa lại.)

Muốn cơ sở dữ liệu thực thi các bất biến, bạn cần khai báo chúng như các *ràng buộc* (*constraint*) trong schema. Ràng buộc khóa ngoại, ràng buộc duy nhất hay ràng buộc kiểm tra (giới hạn giá trị trong một dòng) thường dùng để biểu đạt một số loại bất biến. Những yêu cầu nhất quán phức tạp hơn đôi khi cũng biểu đạt được bằng trigger hoặc materialized view[^12].

Tuy nhiên, các ràng buộc mà cơ sở dữ liệu thường cung cấp có thể khó, thậm chí không thể biểu đạt các bất biến phức tạp. Khi đó, ứng dụng có trách nhiệm định nghĩa giao dịch đúng cách để giữ được tính nhất quán. Nếu ứng dụng ghi dữ liệu sai vi phạm bất biến mà trước đó không khai báo, cơ sở dữ liệu không có cách nào ngăn. Vì vậy, chữ C trong ACID thường phụ thuộc vào cách ứng dụng dùng cơ sở dữ liệu, chứ không phải thuộc tính của riêng cơ sở dữ liệu.

#### Tính cô lập {#sec_transactions_acid_isolation}

Hầu hết cơ sở dữ liệu được nhiều client truy cập cùng lúc. Nếu các client đọc và ghi những phần khác nhau của cơ sở dữ liệu thì không sao, nhưng nếu họ truy cập cùng một bản ghi thì có thể gặp vấn đề đồng thời (race condition).

{{< xref fig="8-1" page="/ch8" anchor="fig_transactions_increment" >}}Hình 8-1{{< /xref >}} là một ví dụ đơn giản về loại vấn đề này. Giả sử hai client cùng tăng một bộ đếm trong cơ sở dữ liệu. Mỗi client phải đọc giá trị hiện tại, cộng 1, rồi ghi giá trị mới ngược lại (giả sử cơ sở dữ liệu không có thao tác tăng sẵn). Trong {{< xref fig="8-1" page="/ch8" anchor="fig_transactions_increment" >}}Hình 8-1{{< /xref >}}, bộ đếm sau hai lần tăng đáng lẽ phải từ 42 lên 44, nhưng do race condition nên chỉ lên 43.

{{< fig num="8-1" id="fig_transactions_increment" src="/fig/ddia_0801.png" caption="Race condition giữa hai client cùng tăng một bộ đếm." class="ddia-figure ddia-figure--panorama" width="2880" height="832" />}}


*Tính cô lập* theo nghĩa ACID là các giao dịch chạy đồng thời được cô lập với nhau: chúng không thể gây nhiễu lẫn nhau. Sách giáo khoa cơ sở dữ liệu kinh điển hình thức hóa tính cô lập thành *serializability*: mỗi giao dịch có thể coi như mình là giao dịch duy nhất đang chạy trên toàn bộ cơ sở dữ liệu. Cơ sở dữ liệu bảo đảm rằng khi các giao dịch đã commit, kết quả giống như chúng chạy *tuần tự* (lần lượt từng cái), dù thực tế chúng có thể chạy đồng thời[^13].

Tuy nhiên, serializability có cái giá về hiệu năng. Trên thực tế, nhiều cơ sở dữ liệu dùng các dạng cô lập yếu hơn serializable, tức cho phép các giao dịch đồng thời gây nhiễu nhau trong một phạm vi nhất định. Một số cơ sở dữ liệu phổ biến, như Oracle, thậm chí không hiện thực hóa serializable: Oracle có mức cô lập tên "serializable", nhưng thực chất hiện thực hóa *snapshot isolation*, một bảo đảm yếu hơn[^10] [^14]. Vì vậy, một số race condition vẫn có thể xảy ra. Ta sẽ xem xét snapshot isolation và các dạng cô lập khác trong ["Các mức cô lập yếu"](/ch8#sec_transactions_isolation_levels).

#### Tính bền vững {#durability}

Mục đích của hệ thống cơ sở dữ liệu là cung cấp nơi lưu trữ an toàn, để bạn yên tâm cất dữ liệu mà không lo mất. *Tính bền vững* (*durability*) là lời hứa rằng một khi giao dịch đã commit thành công, mọi dữ liệu nó ghi sẽ không bị mất, kể cả khi phần cứng hỏng hay cơ sở dữ liệu sập.

Trong cơ sở dữ liệu đơn node, tính bền vững thường nghĩa là dữ liệu đã được ghi vào bộ nhớ không bay hơi như ổ cứng hay SSD. Các thao tác ghi tệp thông thường thường được đệm trong bộ nhớ và gửi xuống đĩa sau, nên có thể mất nếu mất điện đột ngột. Vì vậy nhiều cơ sở dữ liệu dùng system call `fsync()` để bảo đảm dữ liệu thực sự đã xuống đĩa. Cơ sở dữ liệu thường còn có write-ahead log hoặc cơ chế tương tự (xem ["Làm cho B-tree đáng tin cậy"](/ch4#sec_storage_btree_wal)) để khôi phục khi sập giữa chừng lúc ghi.

Trong cơ sở dữ liệu có sao chép (*replication*), tính bền vững có thể nghĩa là dữ liệu đã được sao chép (*replication*) thành công sang một số node nhất định. Cơ sở dữ liệu phải đợi các thao tác ghi hoặc sao chép (*replication*) này hoàn tất rồi mới báo giao dịch commit thành công. Tuy vậy, như đã nói ở ["Độ tin cậy và khả năng chịu lỗi"](/ch2#sec_introduction_reliability), tính bền vững hoàn hảo không tồn tại: nếu mọi ổ đĩa và backup cùng bị hủy, cơ sở dữ liệu hiển nhiên cũng chịu bó tay.


<a id="sidebar_transactions_durability"></a>

> [!TIP] Sao chép (*replication*) và tính bền vững
>
> Trong lịch sử, tính bền vững nghĩa là ghi ra băng lưu trữ; sau đó là ghi ra đĩa hoặc SSD; gần đây, khái niệm này được mở rộng sang sao chép (*replication*). Cách hiện thực nào tốt hơn?
>
> Sự thật là không có phương án nào hoàn hảo:
>
> * Nếu dữ liệu đã ghi xuống đĩa nhưng máy bị sập, dù dữ liệu không mất, bạn cũng không truy cập được cho đến khi sửa máy hoặc chuyển đĩa sang máy khác. Hệ thống có sao chép (*replication*) thì có thể vẫn sẵn sàng.
> * Lỗi tương quan — như mất điện, hoặc một bug khiến mọi node sập khi gặp một đầu vào nhất định — có thể hạ gục mọi bản sao cùng lúc (xem ["Độ tin cậy và khả năng chịu lỗi"](/ch2#sec_introduction_reliability)), làm mất sạch dữ liệu chỉ nằm trong bộ nhớ. Vì vậy, ghi xuống đĩa vẫn quan trọng ngay cả với cơ sở dữ liệu có sao chép (*replication*).
> * Trong hệ thống sao chép bất đồng bộ (*asynchronous replication*), khi leader không còn khả dụng, các thao tác ghi gần nhất có thể bị mất (xem ["Xử lý sự cố node"](/ch6#sec_replication_failover)).
> * Khi mất điện đột ngột, SSD đặc biệt dễ vi phạm bảo đảm mà lẽ ra chúng phải đem lại: ngay cả `fsync` cũng không phải lúc nào cũng hoạt động đúng[^15]. Như mọi phần mềm, firmware của đĩa cũng có bug[^16] [^17]. Ví dụ, có một bug khiến ổ cứng hỏng sau đúng 32.768 giờ hoạt động[^18]. Ngoài ra, `fsync` rất khó dùng đúng. Ngay cả PostgreSQL cũng từng dùng sai suốt hơn 20 năm[^19] [^20] [^21].
> * Tương tác tinh vi giữa storage engine và hiện thực của hệ thống tệp có thể gây ra những bug khó truy vết, làm hỏng tệp trên đĩa sau khi sập[^22] [^23]. Lỗi hệ thống tệp trên một bản sao đôi khi còn lan sang các bản sao khác[^24].
> * Dữ liệu trên đĩa có thể dần dần hỏng mà không ai hay[^25]. Nếu hỏng đã diễn ra một thời gian, các bản sao và backup gần đây cũng có thể hỏng theo. Khi đó, bạn chỉ còn cách thử khôi phục dữ liệu từ backup cũ hơn.
> * Một nghiên cứu về SSD cho thấy 30% đến 80% ổ có ít nhất một khối hỏng trong bốn năm đầu sử dụng, và chỉ một phần số khối hỏng này được firmware sửa được[^26]. Ổ cứng cơ học có tỷ lệ sector hỏng thấp hơn nhưng khả năng hỏng hoàn toàn lại cao hơn SSD.
> * SSD đã mòn nặng (trải qua nhiều chu kỳ ghi/xóa) khi mất điện có thể bắt đầu mất dữ liệu sau vài tuần đến vài tháng, tùy nhiệt độ[^27]. Ổ ít mòn hơn thì không nghiêm trọng đến vậy[^28].
>
> Thực tế, không công nghệ nào đem lại bảo đảm tuyệt đối, chỉ có các biện pháp giảm rủi ro như ghi ra đĩa, sao chép (*replication*) sang máy từ xa và backup — chúng có thể và nên được dùng kết hợp. Như mọi khi, với bất kỳ "bảo đảm" lý thuyết nào, bạn nên giữ thái độ hoài nghi vừa phải.

### Thao tác đơn đối tượng và đa đối tượng {#sec_transactions_multi_object}

Tóm lại, trong ACID, tính nguyên tử và tính cô lập mô tả cách cơ sở dữ liệu nên xử lý khi client thực hiện nhiều thao tác ghi trong cùng một giao dịch:

Tính nguyên tử
: Nếu có lỗi giữa chừng khi thực hiện một chuỗi thao tác ghi, giao dịch phải bị abort và các thao tác ghi đã làm phải bị loại bỏ. Nói cách khác, cơ sở dữ liệu cho bạn bảo đảm "được ăn cả, ngã về không", để bạn khỏi phải lo về lỗi từng phần.

Tính cô lập
: Các giao dịch chạy đồng thời không được gây nhiễu lẫn nhau. Ví dụ, nếu một giao dịch thực hiện nhiều thao tác ghi, thì giao dịch khác phải thấy hoặc toàn bộ chúng, hoặc không thấy gì, chứ không được chỉ thấy một phần.

Các định nghĩa này giả định bạn muốn sửa nhiều đối tượng (dòng, tài liệu hay bản ghi) cùng lúc. *Giao dịch đa đối tượng* (*multi-object transaction*) thường cần thiết khi nhiều mẩu dữ liệu phải được giữ đồng bộ. {{< xref fig="8-2" page="/ch8" anchor="fig_transactions_read_uncommitted" >}}Hình 8-2{{< /xref >}} minh họa bằng một ứng dụng email. Để hiển thị số thư chưa đọc của người dùng, bạn có thể chạy truy vấn như sau:

```
SELECT COUNT(*) FROM emails WHERE recipient_id = 2 AND unread_flag = true
```

{{< fig num="8-2" id="fig_transactions_read_uncommitted" src="/fig/ddia_0802.png" caption="Vi phạm tính cô lập: một giao dịch đọc các thao tác ghi chưa commit của giao dịch khác (\"dirty read\")." class="ddia-figure ddia-figure--wide" width="2880" height="1149" />}}


Tuy nhiên, khi có nhiều thư, truy vấn này có thể quá chậm, nên bạn quyết định lưu số thư chưa đọc vào một trường riêng (một dạng phi chuẩn hóa, xem ["Chuẩn hóa, phi chuẩn hóa và phép nối"](/ch3#sec_datamodels_normalization)). Khi đó, mỗi khi có thư mới đến, bạn phải tăng bộ đếm thư chưa đọc, và mỗi khi thư được đánh dấu đã đọc, phải giảm bộ đếm đó.

Trong {{< xref fig="8-2" page="/ch8" anchor="fig_transactions_read_uncommitted" >}}Hình 8-2{{< /xref >}}, người dùng 2 gặp một bất thường: danh sách hộp thư đã hiện một thư chưa đọc, nhưng bộ đếm vẫn bằng 0 vì chưa được tăng. (Nếu bạn thấy số đếm sai trong ứng dụng email chẳng đáng quan tâm, hãy thay số thư chưa đọc bằng số dư tài khoản khách hàng, và thay thư bằng giao dịch thanh toán.) Tính cô lập ngăn được vấn đề này: nó bảo đảm người dùng 2 hoặc thấy cả thư mới được chèn lẫn bộ đếm đã cập nhật, hoặc không thấy cái nào, chứ không bao giờ thấy trạng thái trung gian không nhất quán.

{{< xref fig="8-3" page="/ch8" anchor="fig_transactions_atomicity" >}}Hình 8-3{{< /xref >}} cho thấy vì sao cần tính nguyên tử: nếu có lỗi giữa chừng giao dịch, nội dung hộp thư và bộ đếm chưa đọc có thể lệch nhau. Trong một giao dịch nguyên tử, nếu việc cập nhật bộ đếm thất bại thì giao dịch bị abort và thư đã chèn được rollback.

{{< fig num="8-3" id="fig_transactions_atomicity" src="/fig/ddia_0803.png" caption="Tính nguyên tử bảo đảm rằng nếu có lỗi, mọi thao tác ghi trước đó của giao dịch đều bị hoàn tác, tránh để lại trạng thái không nhất quán." class="ddia-figure ddia-figure--panorama" width="2880" height="763" />}}


Giao dịch đa đối tượng cần một cách để xác định những thao tác đọc và ghi nào thuộc cùng một giao dịch. Trong cơ sở dữ liệu quan hệ, việc này thường dựa vào kết nối TCP từ client đến máy chủ cơ sở dữ liệu: trên cùng một kết nối, mọi thao tác nằm giữa câu lệnh `BEGIN TRANSACTION` và `COMMIT` thuộc cùng một giao dịch. Nếu kết nối TCP bị ngắt, giao dịch phải bị abort.

Mặt khác, nhiều cơ sở dữ liệu phi quan hệ không có cách nhóm các thao tác như vậy. Ngay cả khi có API đa đối tượng (ví dụ kho khóa-giá trị có thể có thao tác *multi-put* cập nhật nhiều khóa cùng lúc), nó cũng không nhất thiết có ngữ nghĩa giao dịch: lệnh có thể chỉ cập nhật một số khóa còn số khác thất bại, để cơ sở dữ liệu ở trạng thái cập nhật dở dang.

#### Ghi đơn đối tượng {#sec_transactions_single_object}

Tính nguyên tử và tính cô lập cũng áp dụng khi thay đổi một đối tượng duy nhất. Ví dụ, giả sử bạn đang ghi một tài liệu JSON 20 KB vào cơ sở dữ liệu:

* Nếu kết nối mạng đứt sau khi 10 KB đầu đã gửi xong, cơ sở dữ liệu có lưu lại một mảnh JSON 10 KB không thể phân tích không?
* Nếu mất điện đột ngột khi cơ sở dữ liệu đang ghi đè giá trị cũ trên đĩa, cuối cùng có thể ghép lẫn giá trị mới và cũ không?
* Nếu client khác đọc tài liệu đó giữa chừng lúc đang ghi, nó có thấy giá trị mới chỉ cập nhật một phần không?

Những câu hỏi này sẽ khiến người ta rối bời. Vì vậy, hầu như mọi storage engine đều cố bảo đảm tính nguyên tử và tính cô lập ở mức một đối tượng (ví dụ một cặp khóa-giá trị) trên một node. Tính nguyên tử có thể hiện thực bằng log và khôi phục sau sập (xem ["Làm cho B-tree đáng tin cậy"](/ch4#sec_storage_btree_wal)), còn tính cô lập có thể hiện thực bằng cách khóa từng đối tượng (mỗi lúc chỉ cho một luồng truy cập đối tượng).

Một số cơ sở dữ liệu còn cung cấp các thao tác nguyên tử phức tạp hơn, như thao tác tăng, giúp khỏi phải chạy chu trình đọc–sửa–ghi như trong {{< xref fig="8-1" page="/ch8" anchor="fig_transactions_increment" >}}Hình 8-1{{< /xref >}}. Một loại khác cũng phổ biến là *ghi có điều kiện* (*conditional write*): thao tác ghi chỉ xảy ra nếu giá trị chưa bị ai khác sửa đồng thời (xem ["Ghi có điều kiện (compare-and-set)"](/ch8#sec_transactions_compare_and_set)). Nó tương tự thao tác compare-and-set hay compare-and-swap (CAS) trong đồng thời bộ nhớ dùng chung.


> [!NOTE]
> Nói chính xác, *tăng nguyên tử* (*atomic increment*) dùng từ "nguyên tử" theo nghĩa của lập trình đa luồng. Trong ngữ cảnh ACID, nó đáng lẽ phải gọi là *tăng cô lập* (*isolated increment*) hoặc *tăng serializable* (*serializable increment*), nhưng thường không ai gọi vậy.


Các thao tác đơn đối tượng này hữu ích vì chúng tránh được lost update khi nhiều client cùng ghi vào một đối tượng (xem ["Ngăn chặn lost update"](/ch8#sec_transactions_lost_update)). Nhưng chúng không phải giao dịch theo nghĩa thông thường. Ví dụ, tính năng "lightweight transaction" của Cassandra và ScyllaDB, hay chế độ "strong consistency" của Aerospike, cung cấp đọc linearizable và ghi có điều kiện trên một đối tượng đơn lẻ (xem ["Linearizability"](/ch10#sec_consistency_linearizability)), nhưng không đưa ra bảo đảm nào cho thao tác trên nhiều đối tượng.

#### Nhu cầu về giao dịch đa đối tượng {#sec_transactions_need}

Ta có thật sự cần giao dịch đa đối tượng không? Liệu có thể hiện thực mọi ứng dụng chỉ với mô hình dữ liệu khóa-giá trị và thao tác đơn đối tượng?

Có những trường hợp chỉ cần chèn, cập nhật hoặc xóa một đối tượng là đủ. Nhưng trong nhiều trường hợp khác, phải điều phối thao tác ghi trên nhiều đối tượng khác nhau:

* Trong mô hình dữ liệu quan hệ, một dòng ở bảng này thường tham chiếu đến dòng ở bảng khác bằng khóa ngoại. Tương tự, trong mô hình dữ liệu đồ thị, một đỉnh trỏ đến các đỉnh khác qua cạnh. Giao dịch đa đối tượng bảo đảm các tham chiếu này luôn hợp lệ: khi chèn nhiều bản ghi tham chiếu lẫn nhau, khóa ngoại phải đúng và cập nhật, nếu không dữ liệu mất ý nghĩa.
* Trong mô hình dữ liệu tài liệu, các trường cần cập nhật cùng nhau thường nằm trong cùng một tài liệu, và tài liệu được coi là một đối tượng đơn, nên cập nhật một tài liệu không cần giao dịch đa đối tượng. Tuy nhiên, cơ sở dữ liệu tài liệu thiếu chức năng join lại khuyến khích phi chuẩn hóa (xem ["Khi nào dùng mô hình nào"](/ch3#sec_datamodels_document_summary)). Khi cần cập nhật thông tin phi chuẩn hóa — như trường hợp trong {{< xref fig="8-2" page="/ch8" anchor="fig_transactions_read_uncommitted" >}}Hình 8-2{{< /xref >}} — bạn phải cập nhật nhiều tài liệu cùng lúc. Giao dịch rất hữu ích ở đây để dữ liệu phi chuẩn hóa không bị lệch nhau.
* Trong cơ sở dữ liệu có chỉ mục phụ (*secondary index*) (hầu hết trừ kho khóa-giá trị thuần túy), mỗi lần thay đổi giá trị cũng phải cập nhật các chỉ mục (*index*). Xét từ góc độ giao dịch, các chỉ mục (*index*) này là những đối tượng cơ sở dữ liệu riêng biệt. Ví dụ, nếu không có cô lập giao dịch, một bản ghi có thể xuất hiện trong chỉ mục (*index*) này nhưng không xuất hiện trong chỉ mục (*index*) kia, vì chỉ mục (*index*) thứ hai chưa được cập nhật (xem ["Sharding và chỉ mục phụ (*secondary index*)"](/ch7#sec_sharding_secondary_indexes)).

Những ứng dụng như vậy vẫn hiện thực được mà không cần giao dịch. Tuy nhiên, thiếu tính nguyên tử thì việc xử lý lỗi phức tạp hơn nhiều, và thiếu tính cô lập có thể gây vấn đề đồng thời. Ta sẽ bàn các vấn đề này trong ["Các mức cô lập yếu"](/ch8#sec_transactions_isolation_levels), và xem xét các phương án thay thế trong ["Dữ liệu dẫn xuất và giao dịch phân tán"](/ch13#sec_future_derived_vs_transactions).

#### Xử lý lỗi và abort {#handling-errors-and-aborts}

Một đặc điểm then chốt của giao dịch là khi gặp lỗi, nó có thể abort và thử lại an toàn. Cơ sở dữ liệu ACID theo nguyên tắc: nếu bảo đảm về tính nguyên tử, tính cô lập hay tính bền vững có nguy cơ bị phá vỡ, cơ sở dữ liệu thà bỏ hẳn giao dịch còn hơn để nó dở dang.

Tuy nhiên, không phải hệ thống nào cũng theo nguyên tắc này. Đặc biệt, các kho dữ liệu dùng sao chép không leader (*leaderless replication*) (xem ["Sao chép không leader (*leaderless replication*)"](/ch6#sec_replication_leaderless)) hoạt động nhiều hơn theo kiểu "cố hết sức". Có thể tóm tắt là: "Cơ sở dữ liệu làm được bao nhiêu thì làm; nếu gặp lỗi, nó sẽ không hoàn tác những gì đã làm." Vì vậy, khôi phục sau lỗi trở thành trách nhiệm của ứng dụng.

Lỗi là điều không thể tránh, nhưng nhiều lập trình viên thích chỉ nghĩ đến đường đi suôn sẻ hơn là các chi tiết phức tạp của xử lý lỗi. Ví dụ, các framework ánh xạ đối tượng–quan hệ (*ORM*) phổ biến như ActiveRecord của Rails và Django không thử lại giao dịch đã abort — lỗi thường lan lên theo call stack dưới dạng exception, đầu vào của người dùng bị bỏ và họ chỉ nhận một thông báo lỗi. Điều này đáng tiếc, vì mục đích của abort chính là cho phép thử lại an toàn.

Dù thử lại giao dịch đã abort là cơ chế xử lý lỗi đơn giản và hiệu quả, nó không hoàn hảo:

* Giao dịch có thể thực ra đã thành công, nhưng mạng đứt khi máy chủ báo commit thành công cho client, khiến client chờ quá hạn. Thử lại lúc này sẽ khiến giao dịch chạy hai lần, trừ khi ứng dụng có cơ chế khử trùng lặp riêng.
* Nếu lỗi do quá tải hoặc do tranh chấp nghiêm trọng giữa các giao dịch đồng thời, thử lại chỉ làm tình hình tệ hơn. Để tránh vòng phản hồi này, bạn có thể giới hạn số lần thử lại, dùng exponential backoff, và xử lý lỗi liên quan đến quá tải khác với các lỗi còn lại (xem ["Khi hệ thống quá tải không thể phục hồi"](/ch2#sidebar_metastable)).
* Chỉ nên thử lại với lỗi tạm thời (như deadlock, vi phạm cô lập, mạng gián đoạn tạm thời hay failover), còn với lỗi vĩnh viễn (như vi phạm ràng buộc) thì thử lại vô nghĩa.
* Nếu giao dịch còn gây tác dụng phụ bên ngoài cơ sở dữ liệu, tác dụng phụ đó có thể vẫn xảy ra dù giao dịch bị abort. Ví dụ, nếu bạn gửi email, rõ ràng bạn không muốn gửi lại email mỗi lần thử lại giao dịch. Để bảo đảm nhiều hệ thống khác nhau cùng commit hoặc cùng abort, two-phase commit có thể giúp (xem ["Commit hai pha (2PC)"](/ch8#sec_transactions_2pc)).
* Nếu tiến trình client sập trong lúc thử lại, dữ liệu mà nó định ghi vào cơ sở dữ liệu sẽ mất.



## Các mức cô lập yếu {#sec_transactions_isolation_levels}

Nếu hai giao dịch không truy cập cùng dữ liệu, hoặc cả hai đều chỉ đọc, chúng có thể chạy song song an toàn vì không phụ thuộc nhau. Vấn đề đồng thời (race condition) chỉ xuất hiện khi một giao dịch đọc dữ liệu đang bị giao dịch khác sửa đồng thời, hoặc khi hai giao dịch cố sửa cùng dữ liệu một lúc.

Lỗi đồng thời rất khó tìm ra bằng kiểm thử, vì chúng chỉ kích hoạt khi thời điểm xảy ra không thuận lợi. Vấn đề về thời điểm này có thể rất hiếm gặp và thường khó tái hiện. Bản thân hành vi đồng thời cũng khó suy luận, nhất là trong ứng dụng lớn, nơi bạn chưa chắc biết còn đoạn code nào khác đang truy cập cơ sở dữ liệu. Phát triển ứng dụng với một người dùng mỗi lúc đã khó, có nhiều người dùng đồng thời còn khó hơn, vì bất kỳ dữ liệu nào cũng có thể thay đổi bất ngờ bất cứ lúc nào.

Chính vì vậy, từ lâu cơ sở dữ liệu đã cố giấu vấn đề đồng thời khỏi lập trình viên ứng dụng bằng *cô lập giao dịch* (*transaction isolation*). Về lý thuyết, cô lập cho phép bạn giả vờ rằng không hề có đồng thời: cô lập *serializable* nghĩa là cơ sở dữ liệu bảo đảm kết quả của các giao dịch giống như khi chúng chạy *tuần tự* (từng cái một, không đồng thời).

Đáng tiếc, trong thực tế cô lập không đơn giản như vậy. Cô lập serializable có cái giá về hiệu năng, và nhiều cơ sở dữ liệu không muốn trả[^10]. Vì thế, các hệ thống thường dùng mức cô lập yếu hơn, chỉ chống *một số* chứ không phải mọi vấn đề đồng thời. Các mức cô lập này khó hiểu hơn và có thể dẫn đến lỗi tinh vi, nhưng vẫn được dùng rộng rãi trong thực tế[^29].

Các lỗi đồng thời do cô lập giao dịch yếu gây ra không phải chuyện lý thuyết: chúng từng gây thiệt hại tiền bạc khổng lồ[^30] [^31] [^32], khiến kiểm toán viên tài chính phải vào cuộc điều tra[^33], và làm hỏng dữ liệu khách hàng[^34]. Khi những vấn đề này lộ ra, người ta hay nói: "Xử lý dữ liệu tài chính thì phải dùng cơ sở dữ liệu ACID!" Nhưng như vậy là chưa trúng vấn đề. Ngay cả nhiều cơ sở dữ liệu quan hệ phổ biến thường được xem là "ACID" cũng dùng cô lập yếu, nên không nhất thiết ngăn được những vấn đề này.


> [!NOTE]
> Nhân tiện, một phần lớn hệ thống ngân hàng dựa vào các tệp văn bản trao đổi qua FTP bảo mật[^35]. Trong môi trường như vậy, dấu vết kiểm toán và một số biện pháp chống gian lận có sự tham gia của con người thực ra quan trọng hơn các thuộc tính ACID.


Những ví dụ này cũng nhấn mạnh một điểm: ngay cả khi vấn đề đồng thời hiếm gặp trong vận hành bình thường, bạn vẫn phải tính đến khả năng kẻ tấn công cố ý gửi tới API một loạt yêu cầu đồng thời cao để khai thác các lỗi đồng thời[^30]. Do đó, để xây dựng ứng dụng đáng tin cậy và an toàn, bạn phải phòng chống loại lỗi này một cách có hệ thống.

Mục này xem xét một số mức cô lập yếu (tức không serializable) đang dùng trong thực tế, bàn chi tiết race condition nào có thể và không thể xảy ra, để bạn quyết định mức nào phù hợp với ứng dụng. Sau đó ta sẽ đi sâu vào serializable (xem ["Serializable"](/ch8#sec_transactions_serializability)). Ở đây ta minh họa không hình thức bằng ví dụ. Định nghĩa chặt chẽ và phân tích tính chất của chúng có trong tài liệu học thuật[^36] [^37] [^38] [^39].

### Read committed {#sec_transactions_read_committed}

Mức cô lập giao dịch cơ bản nhất là *read committed*, đem lại hai bảo đảm:

1. Khi đọc từ cơ sở dữ liệu, bạn chỉ thấy dữ liệu đã commit (không có *dirty read*).
2. Khi ghi vào cơ sở dữ liệu, bạn chỉ ghi đè lên dữ liệu đã commit (không có *dirty write*).

Một số cơ sở dữ liệu còn hỗ trợ mức cô lập yếu hơn là *read uncommitted*. Nó ngăn dirty write nhưng không ngăn dirty read. Ta sẽ bàn chi tiết hai bảo đảm này.

#### Không có dirty read {#no-dirty-reads}

Hãy hình dung một giao dịch đã ghi dữ liệu vào cơ sở dữ liệu nhưng chưa commit hay abort. Giao dịch khác có thấy dữ liệu chưa commit đó không? Nếu có, đó là *dirty read*[^3].

Giao dịch chạy ở mức cô lập read committed phải ngăn dirty read. Nghĩa là mọi thao tác ghi của một giao dịch chỉ hiện ra cho các giao dịch khác cùng lúc khi giao dịch đó commit. Trong {{< xref fig="8-4" page="/ch8" anchor="fig_transactions_read_committed" >}}Hình 8-4{{< /xref >}}, người dùng 1 đã đặt *x* = 3, nhưng vì giao dịch của họ chưa commit nên *get x* của người dùng 2 vẫn trả về giá trị cũ là 2.

{{< fig num="8-4" id="fig_transactions_read_committed" src="/fig/ddia_0804.png" caption="Không có dirty read: người dùng 2 chỉ thấy giá trị mới của x sau khi giao dịch của người dùng 1 commit." class="ddia-figure ddia-figure--panorama" width="2880" height="832" />}}

Ngăn dirty read có một số lợi ích:

* Nếu giao dịch cập nhật nhiều dòng, dirty read có thể khiến giao dịch khác chỉ thấy một phần các cập nhật. Chẳng hạn trong {{< xref fig="8-2" page="/ch8" anchor="fig_transactions_read_uncommitted" >}}Hình 8-2{{< /xref >}}, người dùng thấy thư chưa đọc mới đến nhưng không thấy bộ đếm đã cập nhật. Đó là dirty read đối với thư. Cơ sở dữ liệu ở trạng thái cập nhật dở dang không chỉ gây khó hiểu mà còn có thể khiến giao dịch khác đưa ra quyết định sai.
* Nếu giao dịch abort, mọi thao tác ghi của nó phải được rollback (như trong {{< xref fig="8-3" page="/ch8" anchor="fig_transactions_atomicity" >}}Hình 8-3{{< /xref >}}). Cho phép dirty read nghĩa là giao dịch có thể đọc phải dữ liệu về sau bị rollback, tức chưa bao giờ thật sự được commit vào cơ sở dữ liệu. Mọi giao dịch từng đọc dữ liệu chưa commit cũng phải abort theo, gây ra cái gọi là *hủy bỏ dây chuyền* (*cascading abort*).

#### Không có dirty write {#sec_transactions_dirty_write}

Điều gì xảy ra nếu hai giao dịch đồng thời cập nhật cùng một dòng trong cơ sở dữ liệu? Ta không biết thứ tự các thao tác ghi, nhưng thường cho rằng thao tác ghi sau sẽ ghi đè thao tác ghi trước.

Nhưng nếu thao tác ghi trước thuộc một giao dịch chưa commit, và thao tác ghi sau ghi đè lên giá trị chưa commit đó thì sao? Đó là *dirty write*[^36]. Giao dịch chạy ở mức read committed phải ngăn dirty write. Cách thường dùng là hoãn thao tác ghi thứ hai cho đến khi giao dịch thực hiện thao tác ghi thứ nhất commit hoặc abort.

Ngăn dirty write tránh được một số vấn đề đồng thời:

* Nếu giao dịch cập nhật nhiều dòng, dirty write có thể gây hậu quả rất tệ. {{< xref fig="8-5" page="/ch8" anchor="fig_transactions_dirty_writes" >}}Hình 8-5{{< /xref >}} minh họa một trang web mua bán xe cũ, nơi Aaliyah và Bryce cùng lúc mua một chiếc xe. Mua xe cần hai thao tác ghi cơ sở dữ liệu: cập nhật tin đăng trên trang sang tên người mua, và gửi hóa đơn bán hàng cho người mua. Trong tình huống ở {{< xref fig="8-5" page="/ch8" anchor="fig_transactions_dirty_writes" >}}Hình 8-5{{< /xref >}}, xe được bán cho Bryce (cập nhật bảng `listings` của anh ta thắng) nhưng hóa đơn lại gửi cho Aaliyah (cập nhật bảng `invoices` của cô ta thắng). Read committed ngăn được sự cố này.
* Tuy nhiên, read committed *không* ngăn được race condition giữa hai lần tăng bộ đếm trong {{< xref fig="8-1" page="/ch8" anchor="fig_transactions_increment" >}}Hình 8-1{{< /xref >}}. Ở đó, thao tác ghi thứ hai xảy ra sau khi giao dịch thứ nhất đã commit, nên không phải dirty write. Kết quả vẫn sai, nhưng vì lý do khác. ["Ngăn chặn lost update"](/ch8#sec_transactions_lost_update) sẽ giải thích cách tăng bộ đếm như vậy một cách an toàn.

{{< fig num="8-5" id="fig_transactions_dirty_writes" src="/fig/ddia_0805.png" caption="Với dirty write, các thao tác ghi xung đột từ những giao dịch khác nhau có thể bị trộn lẫn." class="ddia-figure ddia-figure--wide" width="2658" height="1200" />}}


#### Hiện thực hóa read committed {#sec_transactions_read_committed_impl}

Read committed là mức cô lập rất phổ biến, và là cài đặt mặc định của nhiều cơ sở dữ liệu như Oracle Database, PostgreSQL và SQL Server[^10].

Cách phổ biến nhất để cơ sở dữ liệu ngăn dirty write là dùng khóa ở mức dòng: khi giao dịch muốn sửa một dòng (hoặc đối tượng khác như tài liệu), nó phải lấy khóa của dòng đó trước và giữ đến khi commit hoặc abort. Mỗi dòng chỉ có một giao dịch giữ khóa tại một thời điểm. Nếu giao dịch khác muốn ghi vào cùng dòng, nó phải đợi giao dịch trước commit hoặc abort rồi mới lấy được khóa và tiếp tục. Ở chế độ read committed (hoặc mức cô lập mạnh hơn), cơ sở dữ liệu tự động thực hiện việc khóa này.

Vậy làm sao ngăn dirty read? Một cách là dùng chính khóa đó: yêu cầu mọi giao dịch muốn đọc một dòng đều lấy khóa trong chốc lát, đọc xong thì thả ra ngay. Như vậy, nếu dòng đang chứa giá trị dirty chưa commit thì không đọc được, vì giao dịch đang ghi vẫn giữ khóa.

Tuy nhiên, yêu cầu thao tác đọc phải lấy khóa không hiệu quả trong thực tế. Một giao dịch ghi chạy lâu có thể buộc nhiều giao dịch chỉ đọc phải đợi đến khi nó xong. Điều này không chỉ hại thời gian phản hồi của các giao dịch chỉ đọc mà còn hại khả năng vận hành: một phần ứng dụng chậm đi có thể gây hiệu ứng dây chuyền vì chờ khóa, lan sang những phần khác hoàn toàn không liên quan.

Dù vậy, vẫn có cơ sở dữ liệu dùng khóa để ngăn dirty read, ví dụ IBM Db2 và Microsoft SQL Server với thiết lập `read_committed_snapshot=off`[^29].

Cách phổ biến hơn để ngăn dirty read, như minh họa trong {{< xref fig="8-4" page="/ch8" anchor="fig_transactions_read_committed" >}}Hình 8-4{{< /xref >}}, là: với mỗi dòng được ghi, cơ sở dữ liệu giữ cả giá trị cũ đã commit lẫn giá trị mới do giao dịch đang giữ khóa ghi tạo ra. Trong lúc giao dịch đó còn chạy, các giao dịch khác đọc dòng này chỉ nhận giá trị cũ, chỉ khi giá trị mới được commit mới chuyển sang đọc giá trị mới (chi tiết trong ["Kiểm soát đồng thời đa phiên bản (MVCC)"](/ch8#sec_transactions_snapshot_impl)).

### Snapshot isolation và repeatable read {#sec_transactions_snapshot_isolation}

Thoạt nhìn, dễ nghĩ rằng read committed đã đủ mọi thứ ta cần ở giao dịch: nó cho phép abort (cần cho tính nguyên tử), ngăn đọc kết quả dở dang của giao dịch, và ngăn các thao tác ghi đồng thời trộn lẫn. Quả thật, các tính năng này hữu ích, và bảo đảm mà chúng đem lại mạnh hơn nhiều so với hệ thống không có giao dịch.

Tuy nhiên, ở mức cô lập này vẫn có nhiều cách để lỗi đồng thời xuất hiện. {{< xref fig="8-6" page="/ch8" anchor="fig_transactions_item_many_preceders" >}}Hình 8-6{{< /xref >}} cho thấy một vấn đề mà read committed có thể gặp.

{{< fig num="8-6" id="fig_transactions_item_many_preceders" src="/fig/ddia_0806.png" caption="Read skew: Aaliyah quan sát thấy trạng thái không nhất quán của cơ sở dữ liệu." class="ddia-figure ddia-figure--wide" width="2658" height="1200" />}}


Giả sử Aaliyah có 1.000 đô la tiền gửi ở ngân hàng, chia vào hai tài khoản, mỗi tài khoản 500 đô la. Bây giờ có một giao dịch chuyển 100 đô la từ tài khoản này sang tài khoản kia. Nếu cô ấy xem số dư đúng lúc giao dịch chuyển tiền đang xử lý, cô có thể thấy số dư một tài khoản trước khi tiền chuyển đến (500 đô la), rồi thấy số dư tài khoản kia sau khi tiền đã chuyển đi (400 đô la). Với Aaliyah, tổng hai tài khoản chỉ còn 900 đô la — như thể 100 đô la tự nhiên biến mất.

Bất thường này gọi là *read skew*, một dạng của *nonrepeatable read*: nếu sau khi giao dịch chuyển tiền kết thúc, Aaliyah đọc lại số dư tài khoản 1, cô sẽ thấy 600 đô la, khác với giá trị thấy ở truy vấn trước. Cô lập read committed cho phép read skew, vì tại thời điểm Aaliyah đọc số dư từng tài khoản, chúng đúng là đã được commit.


> [!NOTE]
> Đáng tiếc, từ *skew* có nhiều nghĩa: trước đó ta dùng nó để chỉ *khối lượng công việc mất cân bằng có hot spot* (xem ["Khối lượng công việc lệch và giảm nhẹ hot spot"](/ch7#sec_sharding_skew)), còn ở đây nó chỉ *bất thường về thời điểm*.


Với Aaliyah, vấn đề này sẽ không kéo dài: vài giây sau tải lại trang ngân hàng trực tuyến, nhiều khả năng cô sẽ thấy số dư nhất quán. Nhưng có những tình huống không chịu nổi dù chỉ bất nhất tạm thời:

Backup
: Backup cần tạo một bản sao của toàn bộ cơ sở dữ liệu, với cơ sở dữ liệu lớn có thể mất nhiều giờ. Trong lúc backup, cơ sở dữ liệu vẫn liên tục nhận thao tác ghi. Kết quả là một phần backup có thể chứa phiên bản dữ liệu cũ hơn, phần khác lại chứa phiên bản mới hơn. Nếu khôi phục từ một backup như vậy, sự bất nhất tạm thời (như tiền tự biến mất) sẽ trở thành vĩnh viễn.

Truy vấn phân tích và kiểm tra tính toàn vẹn
: Đôi khi bạn cần chạy truy vấn quét phần lớn cơ sở dữ liệu. Truy vấn như vậy phổ biến trong phân tích (xem ["Hệ thống phân tích và hệ thống tác nghiệp"](/ch1#sec_introduction_analytics)), và cũng có thể dùng để kiểm tra tính toàn vẹn định kỳ nhằm xác nhận mọi thứ ổn (tức phát hiện dữ liệu hỏng). Nếu truy vấn quan sát các phần khác nhau của cơ sở dữ liệu ở những thời điểm khác nhau, kết quả trả về rất có thể vô nghĩa.

*Snapshot isolation*[^36] là giải pháp phổ biến nhất cho vấn đề này. Ý tưởng cốt lõi là mỗi giao dịch đọc từ một *snapshot nhất quán* của cơ sở dữ liệu — nghĩa là giao dịch chỉ thấy toàn bộ dữ liệu đã commit trong cơ sở dữ liệu tại lúc nó bắt đầu. Ngay cả khi giao dịch khác sau đó thay đổi dữ liệu, nó vẫn thấy dữ liệu cũ ở thời điểm cụ thể đó.

Snapshot isolation là một lợi ích lớn cho các truy vấn chỉ đọc chạy lâu như backup và phân tích. Nếu dữ liệu mà truy vấn xử lý liên tục thay đổi trong lúc chạy thì rất khó diễn giải kết quả, còn nếu giao dịch thấy một snapshot nhất quán của cơ sở dữ liệu đóng băng tại một thời điểm thì dễ hiểu hơn nhiều.

Snapshot isolation là tính năng phổ biến: PostgreSQL, MySQL với storage engine InnoDB, Oracle, SQL Server… đều hỗ trợ một biến thể của nó, dù hành vi cụ thể mỗi hệ thống một khác[^29] [^40] [^41]. Một số cơ sở dữ liệu như Oracle, TiDB và Aurora DSQL thậm chí coi snapshot isolation là mức cô lập cao nhất của mình.

#### Kiểm soát đồng thời đa phiên bản (MVCC) {#sec_transactions_snapshot_impl}

Như read committed, snapshot isolation thường dùng khóa ghi để ngăn dirty write (xem ["Hiện thực hóa read committed"](/ch8#sec_transactions_read_committed_impl)). Do đó, một giao dịch ghi có thể chặn giao dịch khác ghi cùng dòng. Tuy nhiên, thao tác đọc không cần lấy khóa nào. Xét về hiệu năng, một nguyên tắc then chốt của snapshot isolation là: *đọc không chặn ghi, ghi không chặn đọc*. Nhờ vậy, cơ sở dữ liệu có thể chạy truy vấn đọc dài trên snapshot nhất quán đồng thời vẫn xử lý bình thường các thao tác ghi, mà hai bên không tranh khóa.

Để hiện thực snapshot isolation, cơ sở dữ liệu tổng quát hóa cơ chế ngăn dirty read trong {{< xref fig="8-4" page="/ch8" anchor="fig_transactions_read_committed" >}}Hình 8-4{{< /xref >}}. Thay vì chỉ giữ hai phiên bản cho mỗi dòng (phiên bản đã commit và phiên bản mới ghi đè nhưng chưa commit), cơ sở dữ liệu có thể phải giữ nhiều phiên bản đã commit khác nhau, vì các giao dịch đang chạy khác nhau có thể cần quan sát trạng thái cơ sở dữ liệu ở những thời điểm khác nhau. Vì nhiều phiên bản của cùng một dòng cùng tồn tại, kỹ thuật này gọi là *kiểm soát đồng thời đa phiên bản* (*multi-version concurrency control*, MVCC).

{{< xref fig="8-7" page="/ch8" anchor="fig_transactions_mvcc" >}}Hình 8-7{{< /xref >}} cho thấy PostgreSQL hiện thực snapshot isolation dựa trên MVCC như thế nào[^40] [^42] [^43] (các hiện thực khác tương tự). Khi bắt đầu, mỗi giao dịch nhận một ID giao dịch (`txid`) duy nhất và tăng đơn điệu. Bất kỳ dữ liệu nào giao dịch ghi vào cơ sở dữ liệu đều được gắn ID giao dịch của bên ghi. (Nói chính xác, ID giao dịch của PostgreSQL là số nguyên 32 bit nên sẽ tràn sau khoảng 4 tỷ giao dịch. Tiến trình `vacuum` lo dọn dẹp để việc tràn không ảnh hưởng dữ liệu.)

{{< fig num="8-7" id="fig_transactions_mvcc" src="/fig/ddia_0807.png" caption="Hiện thực snapshot isolation bằng kiểm soát đồng thời đa phiên bản." class="ddia-figure ddia-figure--standard" width="2658" height="2013" />}}


Mỗi dòng trong bảng có trường `inserted_by` chứa ID của giao dịch đã chèn dòng đó vào bảng, và trường `deleted_by` ban đầu để trống. Khi giao dịch xóa một dòng, dòng chưa bị gỡ ngay khỏi cơ sở dữ liệu mà chỉ được đánh dấu xóa bằng cách đặt `deleted_by` thành ID của giao dịch yêu cầu xóa. Đến lúc chắc chắn không giao dịch nào còn truy cập được dữ liệu đã xóa đó, tiến trình thu gom rác của cơ sở dữ liệu mới gỡ các dòng đã đánh dấu xóa và giải phóng không gian.

Bên trong, cập nhật được chuyển thành một thao tác xóa và một thao tác chèn[^44]. Ví dụ, trong {{< xref fig="8-7" page="/ch8" anchor="fig_transactions_mvcc" >}}Hình 8-7{{< /xref >}}, giao dịch 13 trừ 100 đô la từ tài khoản 2, đổi số dư từ 500 đô la thành 400 đô la. Lúc này bảng `accounts` thực chất chứa hai dòng cho tài khoản 2: dòng số dư 500 đô la được giao dịch 13 đánh dấu xóa, và dòng số dư 400 đô la do giao dịch 13 chèn vào.

Mọi phiên bản của một dòng được lưu trong cùng một heap của cơ sở dữ liệu (xem ["Lưu giá trị trong chỉ mục (*index*)"](/ch4#sec_storage_index_heap)), bất kể giao dịch ghi chúng đã commit hay chưa. Các phiên bản của cùng một dòng tạo thành một danh sách liên kết, đi từ phiên bản mới nhất đến cũ nhất hoặc ngược lại, để truy vấn có thể duyệt nội bộ qua mọi phiên bản của dòng[^45] [^46].

#### Quy tắc hiển thị để quan sát snapshot nhất quán {#sec_transactions_mvcc_visibility}

Khi giao dịch đọc cơ sở dữ liệu, hệ thống dựa vào ID giao dịch để quyết định phiên bản dòng nào nhìn thấy được, phiên bản nào không. Nếu định nghĩa cẩn thận các quy tắc hiển thị, cơ sở dữ liệu có thể trình bày một snapshot nhất quán cho ứng dụng. Cách làm đại thể như sau[^43] :

1. Khi mỗi giao dịch bắt đầu, cơ sở dữ liệu liệt kê mọi giao dịch khác đang chạy lúc đó (chưa commit hoặc abort). Mọi thao tác ghi của các giao dịch này đều bị bỏ qua, dù về sau chúng commit. Nhờ vậy ta thấy một snapshot nhất quán, không bị ảnh hưởng bởi các giao dịch khác commit sau đó.
2. Mọi thao tác ghi của giao dịch có ID muộn hơn (tức khởi động sau khi giao dịch hiện tại bắt đầu, nên không nằm trong danh sách giao dịch đang chạy ở trên) đều bị bỏ qua, bất kể chúng đã commit hay chưa.
3. Mọi thao tác ghi của giao dịch đã abort đều bị bỏ qua, bất kể abort xảy ra lúc nào. Nhờ vậy, khi giao dịch abort, không cần xóa ngay các dòng nó ghi khỏi kho lưu trữ, vì quy tắc hiển thị sẽ lọc chúng đi, và tiến trình thu gom rác có thể dọn sau.
4. Mọi thao tác ghi còn lại đều hiển thị với truy vấn của ứng dụng.

Các quy tắc này áp dụng cho cả chèn dòng lẫn xóa dòng. Trong {{< xref fig="8-7" page="/ch8" anchor="fig_transactions_mvcc" >}}Hình 8-7{{< /xref >}}, khi giao dịch 12 đọc tài khoản 2, nó thấy số dư 500 đô la: giao dịch xóa số dư 500 đô la là giao dịch 13, và theo quy tắc 2, giao dịch 12 không thấy thao tác xóa của giao dịch 13; tương tự, số dư 400 đô la được chèn vào cũng chưa hiển thị.

Nói cách khác, một dòng chỉ hiển thị khi thỏa cả hai điều kiện sau:

* Tại lúc giao dịch đọc bắt đầu, giao dịch chèn dòng đó đã commit.
* Dòng không bị đánh dấu xóa, hoặc nếu đã bị đánh dấu xóa thì giao dịch yêu cầu xóa chưa commit tại lúc giao dịch đọc bắt đầu.

Giao dịch chạy lâu có thể tiếp tục dùng cùng một snapshot, đọc những giá trị mà theo các giao dịch khác đã bị ghi đè hoặc xóa từ lâu. Cơ sở dữ liệu không bao giờ cập nhật giá trị tại chỗ mà chèn phiên bản mới ở mỗi thay đổi, nên chỉ với chi phí rất nhỏ cũng cung cấp được snapshot nhất quán.

<a id="sec_transactions_snapshot_indexes"></a>

#### Chỉ mục (*index*) và snapshot isolation {#indexes-and-snapshot-isolation}

Chỉ mục (*index*) hoạt động thế nào trong cơ sở dữ liệu đa phiên bản? Cách phổ biến nhất là mỗi mục chỉ mục (*index*) trỏ tới một phiên bản dòng khớp với nó (phiên bản cũ nhất hoặc mới nhất). Mỗi phiên bản dòng có thể tham chiếu đến phiên bản cũ hơn hoặc mới hơn kế tiếp. Truy vấn dùng chỉ mục (*index*) phải duyệt qua các phiên bản dòng để tìm dòng vừa hiển thị với giao dịch hiện tại, vừa có giá trị thỏa điều kiện truy vấn. Khi thu gom rác gỡ các phiên bản dòng cũ không còn hiển thị với giao dịch nào, nó cũng có thể xóa luôn các mục chỉ mục (*index*) tương ứng.

Nhiều chi tiết hiện thực ảnh hưởng đến hiệu năng của kiểm soát đồng thời đa phiên bản[^45] [^46]. Ví dụ, PostgreSQL có tối ưu để tránh cập nhật chỉ mục (*index*) nếu các phiên bản khác nhau của cùng một dòng nằm vừa trong cùng một trang[^40]. Một số cơ sở dữ liệu khác không lưu bản sao đầy đủ của dòng đã sửa mà chỉ lưu phần khác biệt giữa các phiên bản để tiết kiệm không gian.

CouchDB, Datomic và LMDB dùng một cách khác. Chúng cũng dùng B-tree (xem ["B-tree"](/ch4#sec_storage_b_trees)), nhưng là biến thể *bất biến* (*immutable*) dùng *copy-on-write*: khi cập nhật, chúng không ghi đè trang của cây mà tạo bản sao mới cho mỗi trang bị sửa. Từ trang cha của nó lên đến gốc cây cũng được tạo bản sao và cập nhật để trỏ tới phiên bản mới của trang con. Các trang không bị thao tác ghi ảnh hưởng thì không cần tạo bản sao, và cây mới tiếp tục dùng chung[^47].

Với B-tree bất biến, mỗi giao dịch ghi (hoặc một lô giao dịch) tạo ra một gốc B-tree mới, và một gốc cụ thể chính là snapshot nhất quán của cơ sở dữ liệu tại lúc nó được tạo. Vì các thao tác ghi về sau không thể sửa B-tree hiện có mà chỉ tạo gốc mới, nên không cần lọc dòng theo ID giao dịch. Cách này cũng cần một tiến trình nền thực hiện nén (*compaction*) và thu gom rác.

#### Snapshot isolation, repeatable read và nhầm lẫn về tên gọi {#snapshot-isolation-repeatable-read-and-naming-confusion}

MVCC là kỹ thuật hiện thực thường dùng trong cơ sở dữ liệu, và cũng hay được dùng để hiện thực snapshot isolation. Tuy vậy, các cơ sở dữ liệu khác nhau đôi khi dùng thuật ngữ khác nhau để chỉ cùng một thứ: ví dụ, snapshot isolation trong PostgreSQL gọi là "repeatable read", còn trong Oracle gọi là "serializable"[^29]. Ngược lại, các hệ thống khác nhau cũng dùng cùng một thuật ngữ cho những thứ khác nhau: "repeatable read" của PostgreSQL là snapshot isolation, còn "repeatable read" của MySQL là một hiện thực MVCC có tính nhất quán yếu hơn snapshot isolation[^41].

Sự lộn xộn về tên gọi này xảy ra vì tiêu chuẩn SQL không có khái niệm snapshot isolation. Tiêu chuẩn dựa trên các mức cô lập do System R định nghĩa năm 1975[^3], khi snapshot isolation chưa ra đời. Tiêu chuẩn định nghĩa "repeatable read", nhìn bề ngoài giống snapshot isolation. Snapshot isolation của PostgreSQL đáp ứng yêu cầu của tiêu chuẩn này, nên nó gọi là "repeatable read" và nhờ đó có thể tuyên bố tuân thủ tiêu chuẩn.

Đáng tiếc, định nghĩa các mức cô lập trong tiêu chuẩn SQL có khuyết điểm: mơ hồ, thiếu chính xác, và không độc lập với hiện thực như một tiêu chuẩn đáng lẽ phải vậy[^36]. Nhiều cơ sở dữ liệu tuyên bố hiện thực repeatable read nhưng bảo đảm thực tế lại khác xa nhau, dù về danh nghĩa đã được chuẩn hóa[^29]. Tài liệu nghiên cứu đã đưa ra định nghĩa hình thức của repeatable read[^37] [^38], nhưng hầu hết hiện thực không tuân theo. Càng rối hơn, IBM Db2 còn dùng "repeatable read" để chỉ serializable[^10].

Kết quả là không ai thật sự biết "repeatable read" nghĩa là gì.

### Ngăn chặn lost update {#sec_transactions_lost_update}

Cho đến giờ, read committed và snapshot isolation chủ yếu bảo đảm điều mà giao dịch chỉ đọc có thể thấy khi có thao tác ghi đồng thời. Ta hầu như chưa bàn đến chuyện hai giao dịch ghi đồng thời, ngoài một loại xung đột ghi–ghi cụ thể là dirty write (xem ["Không có dirty write"](/ch8#sec_transactions_dirty_write)).

Giữa các giao dịch ghi đồng thời còn có vài loại xung đột đáng chú ý khác, nổi tiếng nhất là *lost update*. {{< xref fig="8-1" page="/ch8" anchor="fig_transactions_increment" >}}Hình 8-1{{< /xref >}} minh họa vấn đề này qua hai giao dịch cùng tăng một bộ đếm.

Lost update có thể xảy ra khi ứng dụng đọc một giá trị từ cơ sở dữ liệu, sửa nó rồi ghi ngược lại. Quá trình này gọi là *chu trình read-modify-write* (*read-modify-write cycle*). Nếu hai giao dịch cùng chạy chu trình này đồng thời, một thay đổi có thể bị mất, vì lần ghi sau không bao gồm thay đổi của giao dịch trước. (Đôi khi người ta nói lần ghi sau *xóa sạch* lần ghi trước.) Mẫu này xuất hiện trong nhiều tình huống:

* Tăng bộ đếm hoặc cập nhật số dư tài khoản (đọc giá trị hiện tại, tính giá trị mới, ghi giá trị mới ngược lại);
* Sửa cục bộ một giá trị phức tạp, ví dụ thêm phần tử vào danh sách trong tài liệu JSON (phân tích tài liệu, thực hiện thay đổi, ghi tài liệu đã sửa ngược lại);
* Hai người dùng cùng sửa một trang wiki, mỗi người gửi toàn bộ nội dung trang lên máy chủ để lưu thay đổi, từ đó ghi đè nội dung hiện có trong cơ sở dữ liệu.

Vấn đề này rất phổ biến nên người ta đã đưa ra nhiều giải pháp[^48].

#### Thao tác ghi nguyên tử {#atomic-write-operations}

Nhiều cơ sở dữ liệu cung cấp *thao tác ghi nguyên tử* (*atomic write operation*) để ứng dụng khỏi phải tự hiện thực chu trình read-modify-write. Nếu logic nghiệp vụ biểu đạt được bằng các thao tác này, chúng thường là giải pháp tốt nhất. Ví dụ, câu lệnh sau chạy đồng thời an toàn trong hầu hết cơ sở dữ liệu quan hệ:

```sql
UPDATE counters SET value = value + 1 WHERE key = 'foo';
```

Tương tự, các cơ sở dữ liệu tài liệu như MongoDB cung cấp thao tác nguyên tử để ứng dụng sửa cục bộ tài liệu JSON; Redis cung cấp thao tác nguyên tử để sửa cấu trúc dữ liệu như hàng đợi ưu tiên. Không phải thao tác ghi nào cũng dễ biểu diễn thành thao tác nguyên tử — ví dụ, cập nhật trang wiki liên quan đến chỉnh sửa văn bản tùy ý, có thể xử lý bằng các thuật toán nêu trong ["CRDT và operational transformation"](/ch6#sec_replication_crdts) — nhưng hễ dùng được thì thao tác nguyên tử thường là lựa chọn tốt nhất.

Thao tác nguyên tử thường được hiện thực bằng cách lấy *khóa độc quyền* (*exclusive lock*) trên đối tượng khi đọc, để các giao dịch khác không đọc được đối tượng đó cho đến khi cập nhật xong. Cách khác là buộc mọi thao tác nguyên tử chạy trên một luồng duy nhất.

Đáng tiếc, các framework ORM dễ khiến người ta vô tình viết chu trình read-modify-write không an toàn thay vì dùng thao tác nguyên tử do cơ sở dữ liệu cung cấp[^49] [^50] [^51]. Bug sinh ra thường rất tinh vi, khó phát hiện bằng kiểm thử.

#### Khóa tường minh {#explicit-locking}

Nếu thao tác nguyên tử có sẵn trong cơ sở dữ liệu không đáp ứng được, ứng dụng có thể *khóa tường minh* (*explicit locking*) các đối tượng sắp cập nhật, rồi thực hiện chu trình read-modify-write. Giao dịch nào khác cố cập nhật hoặc khóa cùng đối tượng đó đồng thời phải đợi chu trình trước xong.

Lấy game nhiều người chơi làm ví dụ, nhiều người chơi có thể cùng di chuyển một quân cờ. Lúc này thao tác nguyên tử có thể chưa đủ, vì ứng dụng còn phải bảo đảm nước đi hợp lệ theo luật chơi, mà một phần logic đó khó viết thành truy vấn cơ sở dữ liệu một cách hợp lý. Khi đó có thể dùng khóa để ngăn hai người chơi cùng lúc di chuyển một quân cờ, như {{< xref eg="8-1" page="/ch8" anchor="fig_transactions_select_for_update" >}}Ví dụ 8-1{{< /xref >}}.

{{< eg num="8-1" id="fig_transactions_select_for_update" caption="Khóa tường minh các dòng để ngăn lost update" >}}
```sql
BEGIN TRANSACTION;

SELECT * FROM figures
    WHERE name = 'robot' AND game_id = 222
    FOR UPDATE; ❶

-- Kiểm tra nước đi có hợp lệ không, rồi cập nhật
-- vị trí của quân cờ mà câu SELECT trước đó trả về.
UPDATE figures SET position = 'c4' WHERE id = 1234;

COMMIT;
```
{{< /eg >}}

> ❶ Mệnh đề `FOR UPDATE` cho cơ sở dữ liệu biết phải khóa mọi dòng mà truy vấn này trả về.

Cách này dùng được, nhưng phải cân nhắc kỹ logic ứng dụng mới dùng đúng. Rất dễ quên khóa ở đâu đó trong code mà lẽ ra phải có, từ đó đưa race condition vào.

Ngoài ra, khóa nhiều đối tượng còn có nguy cơ *deadlock*: hai hay nhiều giao dịch chờ nhau nhả khóa. Nhiều cơ sở dữ liệu tự động phát hiện deadlock và abort một giao dịch để hệ thống tiếp tục tiến triển. Ứng dụng chỉ cần thử lại giao dịch bị abort đó.

#### Tự động phát hiện lost update {#automatically-detecting-lost-updates}

Thao tác nguyên tử và khóa ngăn lost update bằng cách buộc các chu trình read-modify-write chạy lần lượt. Một hướng khác là cho các chu trình này chạy song song. Nếu bộ quản lý giao dịch phát hiện lost update, nó abort giao dịch tương ứng và buộc nó thử lại toàn bộ chu trình.

Ưu điểm của cách này là cơ sở dữ liệu có thể thực hiện kiểm tra hiệu quả kết hợp với snapshot isolation. Thật vậy, repeatable read của PostgreSQL, serializable của Oracle và snapshot isolation của SQL Server đều tự động phát hiện lost update và abort giao dịch gây xung đột. Còn repeatable read của MySQL/InnoDB thì không phát hiện lost update[^29] [^41]. Một số tác giả cho rằng[^36] [^38], cơ sở dữ liệu chỉ được coi là cung cấp snapshot isolation khi ngăn được lost update. Theo định nghĩa đó, MySQL không cung cấp snapshot isolation.

Tự động phát hiện lost update là tính năng rất tiện, vì code ứng dụng không cần dùng tính năng đặc biệt nào của cơ sở dữ liệu. Lập trình viên có thể quên khóa hoặc quên dùng thao tác nguyên tử và gây ra bug, còn tự động phát hiện thì khó bị bỏ sót. Tuy nhiên, ứng dụng vẫn có trách nhiệm thử lại giao dịch bị abort.

#### Ghi có điều kiện (compare-and-set) {#sec_transactions_compare_and_set}

Các cơ sở dữ liệu không có giao dịch đôi khi cung cấp thao tác *ghi có điều kiện* (đã nhắc ở mục ["Ghi đơn đối tượng"](/ch8#sec_transactions_single_object) phía trên): chỉ cho phép cập nhật nếu giá trị chưa đổi kể từ lần đọc trước, nhờ vậy ngăn được lost update. Nếu giá trị hiện tại không khớp với giá trị đọc trước đó, cập nhật không có hiệu lực và ứng dụng phải thử lại chu trình read-modify-write. Nó tương đương phiên bản cơ sở dữ liệu của lệnh *compare-and-set* hay *compare-and-swap* (CAS) nguyên tử, vốn được nhiều CPU hỗ trợ.

Ví dụ, để ngăn hai người dùng cùng cập nhật một trang wiki, bạn có thể thử cách viết sau, với mong muốn chỉ cập nhật khi nội dung trang không đổi kể từ lúc người dùng bắt đầu sửa.

```sql
-- Có thể an toàn hoặc không, tùy hiện thực của cơ sở dữ liệu
UPDATE wiki_pages SET content = 'new content'
    WHERE id = 1234 AND content = 'old content';
```

Nếu nội dung đã đổi và không còn khớp `'old content'`, cập nhật sẽ không có hiệu lực, vì vậy cần kiểm tra kết quả cập nhật và thử lại nếu cần. Cũng có thể thay vì so sánh toàn bộ nội dung, dùng một cột số phiên bản tăng mỗi lần cập nhật, và chỉ áp dụng cập nhật khi số phiên bản hiện tại chưa đổi. Cách này đôi khi gọi là *khóa lạc quan* (*optimistic locking*)[^52].

Nếu giao dịch khác sửa `content` đồng thời, theo quy tắc hiển thị của MVCC thì nội dung mới có thể không hiển thị (xem ["Quy tắc hiển thị để quan sát snapshot nhất quán"](/ch8#sec_transactions_mvcc_visibility)). Nhiều hiện thực MVCC có ngoại lệ cho tình huống này: giá trị do giao dịch khác ghi, dù không hiển thị trong snapshot, vẫn hiển thị khi tính mệnh đề `WHERE` của các truy vấn `UPDATE` và `DELETE`.

#### Giải quyết xung đột và sao chép (*replication*) {#conflict-resolution-and-replication}

Trong cơ sở dữ liệu có sao chép (*replication*) (xem [Chương 6](/ch6#ch_replication)), ngăn lost update có thêm một khía cạnh: dữ liệu có bản sao trên nhiều node và có thể bị sửa đồng thời ở các node khác nhau, nên cần thêm biện pháp.

Khóa và thao tác ghi có điều kiện đều giả định có một bản sao dữ liệu mới nhất duy nhất. Nhưng các cơ sở dữ liệu dùng sao chép đa leader (*multi-leader replication*) hoặc không leader thường cho phép nhiều thao tác ghi xảy ra đồng thời rồi sao chép bất đồng bộ (*asynchronous replication*) sang node khác, nên không thể bảo đảm có một bản sao mới nhất duy nhất. Vì vậy, các kỹ thuật dựa trên khóa hay ghi có điều kiện không áp dụng được ở đây. (["Linearizability"](/ch10#sec_consistency_linearizability) sẽ bàn sâu hơn vấn đề này.)

Các cơ sở dữ liệu sao chép (*replication*) như vậy thường dùng cách khác, như đã nêu trong ["Xử lý xung đột ghi"](/ch6#sec_replication_write_conflicts): cho phép các thao tác ghi đồng thời tạo ra nhiều phiên bản xung đột của cùng một giá trị (còn gọi là *siblings*), rồi dùng code ứng dụng hoặc cấu trúc dữ liệu đặc biệt giải quyết xung đột và hợp nhất các phiên bản sau.

Nếu các cập nhật có tính giao hoán — tức áp dụng theo thứ tự khác nhau trên các bản sao khác nhau vẫn cho cùng kết quả — thì hợp nhất các giá trị xung đột có thể ngăn lost update. Tăng bộ đếm và thêm phần tử vào tập hợp đều là thao tác giao hoán. Đó chính là ý tưởng đằng sau CRDT, giới thiệu ở ["CRDT và operational transformation"](/ch6#sec_replication_crdts). Tuy nhiên, các thao tác như ghi có điều kiện không thể biến thành giao hoán.

Mặt khác, cách giải quyết xung đột *last write wins* (LWW) rất dễ làm mất cập nhật. Xem chi tiết ở ["Last write wins (loại bỏ các lần ghi đồng thời)"](/ch6#sec_replication_lww). Đáng tiếc, nhiều cơ sở dữ liệu sao chép (*replication*) mặc định dùng LWW.

### Write skew và phantom {#sec_transactions_write_skew}

Phần trên đã giới thiệu *dirty write* và *lost update*: hai race condition có thể xảy ra khi các giao dịch khác nhau ghi đồng thời cùng một đối tượng. Để tránh hỏng dữ liệu, phải ngăn chúng xảy ra — có thể để cơ sở dữ liệu tự động phòng ngừa, hoặc thủ công dùng các biện pháp bảo vệ như khóa, thao tác ghi nguyên tử.

Tuy nhiên, các race condition tiềm ẩn giữa những thao tác ghi đồng thời còn nhiều hơn thế. Mục này sẽ xem xét một số xung đột tinh vi hơn.

Hãy bắt đầu bằng một ví dụ: bạn đang viết ứng dụng để bác sĩ quản lý lịch trực của bệnh viện. Bệnh viện thường muốn lúc nào cũng có vài bác sĩ trực, nhưng ít nhất phải có một người. Bác sĩ có thể bỏ ca của mình (chẳng hạn vì chính họ cũng bị ốm), với điều kiện trong ca đó còn ít nhất một đồng nghiệp tiếp tục trực[^53] [^54].

Giả sử một ca chỉ có hai bác sĩ trực là Aaliyah và Bryce. Cả hai đều thấy không khỏe nên quyết định xin nghỉ, và trùng hợp là họ bấm nút rời ca gần như cùng lúc. Chuyện xảy ra tiếp theo như trong {{< xref fig="8-8" page="/ch8" anchor="fig_transactions_write_skew" >}}Hình 8-8{{< /xref >}}.

{{< fig num="8-8" id="fig_transactions_write_skew" src="/fig/ddia_0808.png" caption="Ví dụ về write skew gây lỗi cho ứng dụng." class="ddia-figure ddia-figure--standard" width="2658" height="1863" />}}


Mỗi giao dịch trước hết kiểm tra hiện có ít nhất hai bác sĩ đang trực không. Nếu có, ứng dụng cho rằng một trong hai có thể rời ca an toàn. Vì cơ sở dữ liệu dùng snapshot isolation, cả hai lần kiểm tra đều trả về `2`, nên cả hai giao dịch đều sang bước sau. Aaliyah cập nhật bản ghi của mình để rời ca, còn Bryce làm điều tương tự. Cả hai giao dịch đều commit thành công, kết quả là không còn bác sĩ nào trực, vi phạm yêu cầu "ít nhất một bác sĩ trực".

#### Đặc điểm của write skew {#characterizing-write-skew}

Bất thường này gọi là *write skew*[^36]. Nó không phải dirty write cũng không phải lost update, vì hai giao dịch cập nhật hai đối tượng khác nhau — bản ghi trực ca của Aaliyah và của Bryce. Xung đột ở đây không rõ ràng như vậy, nhưng đúng là một race condition: nếu hai giao dịch chạy lần lượt, bác sĩ thứ hai sẽ không rời ca được. Hành vi bất thường chỉ xảy ra khi các giao dịch chạy đồng thời.

Có thể xem write skew là sự tổng quát hóa của lost update: write skew có thể xảy ra khi hai giao dịch đọc cùng một tập đối tượng, rồi mỗi giao dịch cập nhật một số trong các đối tượng đó (các giao dịch khác nhau có thể cập nhật các đối tượng khác nhau). Nếu các giao dịch khác nhau tình cờ cập nhật cùng một đối tượng, ta có dirty write hoặc lost update, tùy thời điểm.

Có nhiều cách ngăn lost update, nhưng với write skew thì lựa chọn ít hơn nhiều:

* Thao tác nguyên tử trên một đối tượng không giúp được, vì ở đây có nhiều đối tượng liên quan.
* Một số hiện thực snapshot isolation tự động phát hiện được lost update, nhưng tiếc là với write skew cũng không giúp ích. Repeatable read của PostgreSQL, repeatable read của MySQL/InnoDB, serializable của Oracle và snapshot isolation của SQL Server đều không tự động phát hiện write skew[^29]. Để tự động ngăn write skew, cần cô lập serializable thực sự (xem ["Serializable"](/ch8#sec_transactions_serializability)).
* Một số cơ sở dữ liệu cho phép cấu hình các ràng buộc do chính cơ sở dữ liệu thực thi, như ràng buộc duy nhất, ràng buộc khóa ngoại hoặc giới hạn trên một số giá trị nhất định. Nhưng để biểu đạt "ít nhất một bác sĩ trực" thì cần ràng buộc liên quan đến nhiều đối tượng. Hầu hết cơ sở dữ liệu không hỗ trợ sẵn loại ràng buộc này, nhưng bạn có thể thử hiện thực bằng trigger hoặc materialized view, như đã nói trong ["Tính nhất quán"](/ch8#sec_transactions_acid_consistency)[^12].
* Nếu không dùng được cô lập serializable, lựa chọn tốt thứ nhì ở đây có lẽ là khóa tường minh các dòng mà giao dịch phụ thuộc. Trong ví dụ bác sĩ trực, bạn có thể viết như sau:

    ```sql
    BEGIN TRANSACTION;

    SELECT * FROM doctors
        WHERE on_call = true
        AND shift_id = 1234 FOR UPDATE; ❶

    UPDATE doctors
       SET on_call = false
       WHERE name = 'Aaliyah'
       AND shift_id = 1234;

    COMMIT;
    ```

> ❶ Như trước, `FOR UPDATE` yêu cầu cơ sở dữ liệu khóa mọi dòng mà truy vấn này trả về.

#### Thêm các ví dụ về write skew {#more-examples-of-write-skew}

Thoạt nhìn write skew có vẻ khó hiểu, nhưng một khi đã biết đến nó, bạn sẽ nhận ra nó có thể xuất hiện ở nhiều tình huống. Sau đây là thêm vài ví dụ:

Hệ thống đặt phòng họp
: Giả sử bạn muốn bảo đảm cùng một phòng họp không bị đặt trùng trong cùng một khung giờ[^55]. Khi có người đặt phòng, trước tiên bạn kiểm tra xem có xung đột không (tức cùng phòng có lịch đặt chồng khoảng thời gian không), nếu không thì mới tạo cuộc họp (xem {{< xref eg="8-2" page="/ch8" anchor="fig_transactions_meeting_rooms" >}}Ví dụ 8-2{{< /xref >}}).
    
    {{< eg num="8-2" id="fig_transactions_meeting_rooms" caption="Hệ thống đặt phòng họp cố tránh đặt trùng (không an toàn dưới snapshot isolation)" >}}
    ```sql
    BEGIN TRANSACTION;
    
    -- Kiểm tra xem có lịch đặt nào hiện có chồng lên khung 12 giờ trưa đến 1 giờ chiều không
    SELECT COUNT(*) FROM bookings
    WHERE room_id = 123 AND
    end_time > '2025-01-01 12:00' AND start_time < '2025-01-01 13:00';
    
    -- Nếu truy vấn trước trả về không:
    INSERT INTO bookings (room_id, start_time, end_time, user_id)
    VALUES (123, '2025-01-01 12:00', '2025-01-01 13:00', 666);
    
    COMMIT;
    ```
    {{< /eg >}}

    Đáng tiếc, snapshot isolation không ngăn được người dùng khác đồng thời chèn một cuộc họp xung đột. Để bảo đảm lịch không xung đột, bạn vẫn phải dùng cô lập serializable.

Game nhiều người chơi
: Trong {{< xref eg="8-1" page="/ch8" anchor="fig_transactions_select_for_update" >}}Ví dụ 8-1{{< /xref >}}, ta dùng khóa để ngăn lost update, tức bảo đảm hai người chơi không thể cùng lúc di chuyển một quân cờ. Nhưng khóa này không ngăn được người chơi di chuyển hai quân cờ khác nhau đến cùng một vị trí trên bàn cờ, hay các nước đi vi phạm luật khác. Tùy luật cụ thể, đôi khi dùng được ràng buộc duy nhất, nếu không thì rất dễ xảy ra write skew.

Giành tên người dùng
: Trên website yêu cầu tên người dùng là duy nhất, hai người dùng có thể cùng lúc cố tạo tài khoản với một tên. Có thể dùng giao dịch để kiểm tra trước xem tên đã có người dùng chưa, nếu chưa thì tạo tài khoản với tên đó. Nhưng như các ví dụ trước, cách này không an toàn dưới snapshot isolation. May là ràng buộc duy nhất giải quyết vấn đề này dễ dàng: giao dịch thứ hai cố đăng ký tên đó sẽ bị abort vì vi phạm ràng buộc.

Ngăn chi tiêu trùng
: Dịch vụ cho phép người dùng tiêu tiền hoặc điểm phải bảo đảm người dùng không chi vượt số dư. Một cách hiện thực là chèn một khoản chi tạm vào tài khoản người dùng, rồi liệt kê mọi khoản trong tài khoản và kiểm tra tổng số dư còn dương. Khi xảy ra write skew, hai khoản chi có thể được chèn đồng thời và cộng lại làm số dư âm, mà không giao dịch nào nhận ra giao dịch kia.

#### Phantom gây ra write skew {#sec_transactions_phantom}

Các ví dụ trên đều theo một mẫu tương tự:

1. Truy vấn `SELECT` tìm các dòng thỏa một điều kiện để kiểm tra yêu cầu có được thỏa không (có ít nhất hai bác sĩ đang trực, phòng họp chưa có lịch đặt trong khung giờ đó, ô đích trên bàn cờ chưa có quân nào, tên người dùng chưa bị dùng, tài khoản còn tiền).
2. Tùy kết quả truy vấn đầu, code ứng dụng quyết định bước tiếp: tiếp tục, hoặc báo lỗi cho người dùng và abort.
3. Nếu quyết định tiếp tục, ứng dụng ghi vào cơ sở dữ liệu (`INSERT`, `UPDATE` hoặc `DELETE`) rồi commit giao dịch.

Thao tác ghi này làm thay đổi tiền đề mà quyết định ở bước 2 dựa vào. Nói cách khác, nếu sau khi ghi commit mà chạy lại truy vấn `SELECT` ở bước 1, kết quả sẽ khác, vì tập dòng thỏa điều kiện tìm kiếm đã đổi (ít hơn một bác sĩ trực, phòng họp đã có lịch trong khung giờ đó, ô đích trên bàn cờ đã bị quân vừa di chuyển chiếm, tên người dùng đã bị dùng, hoặc số dư tài khoản đã giảm).

Các bước này cũng có thể theo thứ tự khác. Ví dụ, có thể ghi trước, rồi chạy truy vấn `SELECT`, cuối cùng dựa vào kết quả truy vấn để quyết định abort hay commit.

Trong ví dụ bác sĩ trực, bước 3 sửa một dòng mà bước 1 trả về, nên ta có thể khóa các dòng ở bước 1 (`SELECT FOR UPDATE`) để giao dịch an toàn và tránh write skew. Nhưng bốn ví dụ còn lại thì khác: chúng kiểm tra sự *không tồn tại* của dòng thỏa một điều kiện, và thao tác ghi sau đó *thêm* một dòng thỏa chính điều kiện ấy. Nếu truy vấn ở bước 1 không trả về dòng nào, `SELECT FOR UPDATE` không có gì để khóa[^56].

Hiện tượng thao tác ghi của một giao dịch làm thay đổi kết quả truy vấn tìm kiếm trong giao dịch khác gọi là *phantom*[^4]. Snapshot isolation tránh được phantom trong truy vấn chỉ đọc, nhưng với các giao dịch đọc–ghi như trên, phantom có thể dẫn đến write skew đặc biệt khó chơi. SQL do ORM sinh ra cũng dễ gặp write skew[^50] [^51].

#### Vật chất hóa xung đột {#materializing-conflicts}

Nếu vấn đề của phantom là không có đối tượng nào để gắn khóa, có lẽ ta có thể chủ động đưa vào cơ sở dữ liệu một đối tượng khóa nhân tạo?

Lấy đặt phòng họp làm ví dụ, bạn có thể tạo một bảng phòng và khung giờ. Mỗi dòng trong bảng ứng với một khung giờ cụ thể (chẳng hạn 15 phút) của một phòng cụ thể, và các dòng này được tạo sẵn cho mọi tổ hợp phòng và khung giờ có thể có trong sáu tháng tới.

Bây giờ, giao dịch tạo lịch đặt có thể khóa (`SELECT FOR UPDATE`) các dòng trong bảng ứng với phòng và khung giờ mong muốn. Sau khi lấy được khóa, nó kiểm tra lịch đặt chồng chéo và chèn lịch đặt mới như bình thường. Lưu ý rằng bảng phụ này không lưu thông tin đặt phòng — nó chỉ thuần túy là một tập khóa để ngăn các lịch đặt trong cùng phòng, cùng khoảng thời gian bị sửa đồng thời.

Cách này gọi là *vật chất hóa xung đột* (*materializing conflicts*): nó biến phantom thành xung đột khóa trên một tập dòng cụ thể trong cơ sở dữ liệu[^14]. Đáng tiếc, việc vật chất hóa xung đột vừa khó thiết kế vừa dễ sai, và để cơ chế kiểm soát đồng thời len vào mô hình dữ liệu của ứng dụng cũng chẳng đẹp đẽ gì. Vì vậy, vật chất hóa xung đột chỉ nên là giải pháp cuối cùng khi không còn lựa chọn nào khác. Trong phần lớn trường hợp, cô lập serializable tốt hơn nhiều.



## Serializable {#sec_transactions_serializability}

Chương này đã giới thiệu vài loại giao dịch dễ gặp race condition. Read committed và snapshot isolation ngăn được một số trong đó, nhưng không ngăn được loại khác, trong đó write skew và phantom đặc biệt khó chơi. Tình cảnh hiện tại thật sự đáng nản:

* Các mức cô lập khó hiểu, và hiện thực giữa các cơ sở dữ liệu không nhất quán (ví dụ ý nghĩa của "repeatable read" khác nhau rất nhiều).
* Chỉ nhìn code ứng dụng, rất khó biết nó chạy ở một mức cô lập cụ thể có an toàn không. Với ứng dụng lớn thì càng vậy, vì bạn chưa chắc biết những thao tác nào khác có thể xảy ra đồng thời.
* Không có công cụ tốt giúp phát hiện race condition. Về nguyên tắc, phân tích tĩnh có thể có ích[^33], nhưng các kết quả nghiên cứu chưa đi vào thực tế. Vấn đề đồng thời thường không tất định, chỉ xuất hiện khi thời điểm xảy ra không thuận lợi, nên cũng khó phát hiện bằng kiểm thử.

Đây không phải vấn đề mới — tình hình vẫn như vậy kể từ khi các mức cô lập yếu được đưa ra vào thập niên 1970[^3]. Câu trả lời của giới nghiên cứu luôn đơn giản: hãy dùng cô lập *serializable*!

Serializable là mức cô lập mạnh nhất. Nó bảo đảm rằng dù các giao dịch thực tế chạy song song, kết quả cuối cùng vẫn giống như khi chúng chạy *tuần tự* (lần lượt từng cái, không đồng thời). Vì vậy, nếu giao dịch chạy riêng lẻ cho kết quả đúng, cơ sở dữ liệu bảo đảm chúng vẫn đúng khi chạy đồng thời. Nói cách khác, cơ sở dữ liệu ngăn được *mọi* race condition có thể xảy ra.

Nếu serializable vượt trội sự hỗn loạn của cô lập yếu như vậy, sao không phải ai cũng dùng? Để trả lời, ta cần xem serializable được hiện thực ra sao và hiệu năng thế nào. Hầu hết cơ sở dữ liệu hiện cung cấp serializable dùng một trong ba kỹ thuật sau, mà phần còn lại của chương sẽ lần lượt bàn:

* Thực thi giao dịch theo đúng thứ tự tuần tự (xem ["Thực thi tuần tự thực sự"](/ch8#sec_transactions_serial));
* Khóa hai pha (xem ["Khóa hai pha (2PL)"](/ch8#sec_transactions_2pl)), vốn suốt nhiều thập kỷ là lựa chọn khả thi duy nhất;
* Các kỹ thuật kiểm soát đồng thời lạc quan như serializable snapshot isolation (xem ["Serializable snapshot isolation (SSI)"](/ch8#sec_transactions_ssi)).

### Thực thi tuần tự thực sự {#sec_transactions_serial}

Cách đơn giản nhất để tránh vấn đề đồng thời là loại bỏ hẳn đồng thời: thực thi mỗi lần một giao dịch, theo thứ tự tuần tự, trên một luồng duy nhất. Làm vậy ta hoàn toàn né được việc phát hiện và ngăn xung đột giữa các giao dịch, và mức cô lập thu được, theo định nghĩa, là serializable.

Ý tưởng này nghe hiển nhiên, nhưng mãi đến thập niên 2000, những người thiết kế cơ sở dữ liệu mới kết luận rằng dùng vòng lặp đơn luồng để thực thi giao dịch là khả thi[^57]. Suốt 30 năm trước đó, đa luồng đồng thời được xem là điều kiện cần để đạt hiệu năng tốt. Vậy điều gì đã thay đổi để thực thi đơn luồng trở nên khả thi?

Hai tiến triển đã thúc đẩy việc nghĩ lại này:

* RAM đã đủ rẻ để nhiều trường hợp có thể đưa toàn bộ tập dữ liệu đang hoạt động vào bộ nhớ (xem ["Lưu trữ toàn bộ trong bộ nhớ"](/ch4#sec_storage_inmemory)). Khi mọi dữ liệu giao dịch cần đều nằm trong bộ nhớ, giao dịch chạy nhanh hơn nhiều so với việc chờ nạp dữ liệu từ đĩa.
* Những người thiết kế cơ sở dữ liệu nhận ra rằng giao dịch OLTP thường ngắn và chỉ thực hiện ít thao tác đọc/ghi (xem ["Hệ thống phân tích và hệ thống tác nghiệp"](/ch1#sec_introduction_analytics)). Ngược lại, truy vấn phân tích chạy lâu thường chỉ đọc, nên có thể chạy trên snapshot nhất quán (dùng snapshot isolation) bên ngoài vòng lặp thực thi tuần tự.

Các hệ thống như VoltDB/H-Store, Redis và Datomic thực thi giao dịch tuần tự[^58] [^59] [^60]. Hệ thống thiết kế cho thực thi đơn luồng đôi khi lại nhanh hơn hệ thống hỗ trợ đồng thời, vì nó tránh được chi phí điều phối của việc khóa. Tuy nhiên, thông lượng của nó bị giới hạn ở một lõi CPU. Để tận dụng tối đa luồng đơn này, giao dịch phải được tổ chức khác với dạng truyền thống.

#### Gói giao dịch trong thủ tục lưu trữ {#encapsulating-transactions-in-stored-procedures}

Thuở sơ khai của cơ sở dữ liệu, người ta hình dung giao dịch có thể bao trọn toàn bộ luồng hoạt động của người dùng. Ví dụ, đặt vé máy bay phải qua nhiều giai đoạn: tìm đường bay, giá vé và chỗ trống; chốt hành trình; đặt chỗ cho từng chặng; nhập thông tin hành khách; và cuối cùng thanh toán. Những người thiết kế cơ sở dữ liệu cho rằng sẽ rất lý tưởng nếu cả luồng này được commit nguyên tử (*atomic commitment*) như một giao dịch.

Đáng tiếc, con người ra quyết định và phản hồi rất chậm. Nếu giao dịch cơ sở dữ liệu phải chờ người dùng nhập liệu, cơ sở dữ liệu sẽ phải hỗ trợ một lượng giao dịch đồng thời có thể cực lớn, mà phần lớn đang nhàn rỗi. Hầu hết cơ sở dữ liệu không làm được việc đó hiệu quả, nên gần như mọi ứng dụng OLTP đều tránh chờ người dùng tương tác bên trong giao dịch để giữ giao dịch ngắn. Với ứng dụng web, điều này nghĩa là giao dịch phải commit trong cùng một yêu cầu HTTP, không kéo dài qua nhiều yêu cầu, còn yêu cầu HTTP mới sẽ bắt đầu giao dịch mới.

Ngay cả khi đã đưa con người ra khỏi đường găng, giao dịch vẫn thường dùng mô hình client/server tương tác, thực thi từng câu lệnh một. Ứng dụng gửi truy vấn, đọc kết quả, rồi tùy kết quả truy vấn đầu mà quyết định có gửi truy vấn tiếp theo không, cứ thế lặp lại. Truy vấn và kết quả liên tục đi qua lại giữa máy chạy code ứng dụng và máy chủ cơ sở dữ liệu.

Giao dịch tương tác như vậy tốn rất nhiều thời gian cho giao tiếp mạng giữa ứng dụng và cơ sở dữ liệu. Nếu cơ sở dữ liệu cấm đồng thời và mỗi lần chỉ xử lý một giao dịch thì thông lượng sẽ thấp khủng khiếp: phần lớn thời gian cơ sở dữ liệu chờ ứng dụng gửi truy vấn kế tiếp của giao dịch hiện tại. Cơ sở dữ liệu dùng mô hình này buộc phải xử lý nhiều giao dịch đồng thời mới đạt hiệu năng hợp lý.

Vì vậy, các hệ thống xử lý giao dịch tuần tự đơn luồng không cho phép giao dịch tương tác nhiều câu lệnh. Ứng dụng hoặc chỉ dùng giao dịch một câu lệnh, hoặc nộp trước toàn bộ code của giao dịch cho cơ sở dữ liệu dưới dạng *thủ tục lưu trữ* (*stored procedure*)[^61].

Sự khác biệt giữa giao dịch tương tác và thủ tục lưu trữ được minh họa trong {{< xref fig="8-9" page="/ch8" anchor="fig_transactions_stored_proc" >}}Hình 8-9{{< /xref >}}. Miễn là mọi dữ liệu giao dịch cần đều nằm trong bộ nhớ, thủ tục lưu trữ chạy rất nhanh mà không phải chờ I/O mạng hay đĩa.

{{< fig num="8-9" id="fig_transactions_stored_proc" src="/fig/ddia_0809.png" caption="Sự khác biệt giữa giao dịch tương tác và thủ tục lưu trữ (dùng giao dịch ví dụ trong Hình 8-8)." class="ddia-figure ddia-figure--wide" width="2658" height="1535" />}}

#### Ưu và nhược điểm của thủ tục lưu trữ {#sec_transactions_stored_proc_tradeoffs}

Thủ tục lưu trữ đã tồn tại lâu trong cơ sở dữ liệu quan hệ, và là một phần của chuẩn SQL (SQL/PSM) từ năm 1999. Tuy nhiên, chúng mang tiếng xấu vì nhiều lý do:

* Theo truyền thống, mỗi hãng cơ sở dữ liệu có ngôn ngữ thủ tục lưu trữ riêng (Oracle dùng PL/SQL, SQL Server dùng T-SQL, PostgreSQL dùng PL/pgSQL…). Các ngôn ngữ này không theo kịp sự phát triển của ngôn ngữ lập trình đa dụng, ngày nay chúng khá xấu xí, cũ kỹ, và thiếu hệ sinh thái thư viện mà hầu hết ngôn ngữ lập trình đều có.
* Code chạy trong cơ sở dữ liệu khó quản lý: so với máy chủ ứng dụng, nó khó gỡ lỗi hơn, bất tiện khi quản lý phiên bản và triển khai, kiểm thử phiền hơn, và khó tích hợp với hệ thống giám sát thu thập metric.
* Cơ sở dữ liệu thường nhạy cảm về hiệu năng hơn máy chủ ứng dụng, vì một instance cơ sở dữ liệu thường được nhiều máy chủ ứng dụng dùng chung. Một thủ tục lưu trữ viết kém (ví dụ chiếm nhiều bộ nhớ hoặc thời gian CPU) có thể gây rắc rối lớn hơn đoạn code tệ tương tự trên máy chủ ứng dụng.
* Trong hệ thống multitenant cho phép tenant tự viết thủ tục lưu trữ, việc chạy code không đáng tin cậy trong cùng tiến trình với nhân cơ sở dữ liệu gây rủi ro bảo mật[^62].

Tuy nhiên, những vấn đề này không phải không vượt qua được. Các hiện thực thủ tục lưu trữ hiện đại đã bỏ PL/SQL và chuyển sang dùng ngôn ngữ lập trình đa dụng sẵn có: VoltDB dùng Java hoặc Groovy, Datomic dùng Java hoặc Clojure, Redis dùng Lua, MongoDB dùng JavaScript.

Thủ tục lưu trữ cũng hữu ích khi logic ứng dụng không tiện nhúng ở nơi khác. Chẳng hạn, ứng dụng dùng GraphQL có thể mở trực tiếp cơ sở dữ liệu qua một proxy GraphQL. Nếu proxy không hỗ trợ logic xác thực phức tạp, bạn có thể nhúng logic đó thẳng vào cơ sở dữ liệu bằng thủ tục lưu trữ, còn nếu cơ sở dữ liệu không hỗ trợ thủ tục lưu trữ, bạn đành triển khai riêng một dịch vụ xác thực giữa proxy và cơ sở dữ liệu.

Thủ tục lưu trữ kết hợp với dữ liệu trong bộ nhớ khiến việc thực thi mọi giao dịch trên một luồng duy nhất trở thành phương án khả thi. Miễn là thủ tục lưu trữ không phải chờ I/O và tránh được chi phí của các cơ chế kiểm soát đồng thời khác, luồng đơn cũng đạt được thông lượng khá lớn.

VoltDB còn dùng thủ tục lưu trữ để sao chép (*replication*): thay vì sao chép (*replication*) các thao tác ghi của giao dịch từ node này sang node khác, nó chạy cùng một thủ tục lưu trữ trên mỗi bản sao. Do đó VoltDB đòi hỏi thủ tục lưu trữ phải *tất định* (*deterministic*), tức khi chạy trên các node khác nhau phải cho cùng kết quả. Ví dụ, nếu giao dịch cần dùng ngày giờ hiện tại, nó phải lấy qua một API tất định đặc biệt (xem thêm về thao tác tất định ở ["Durable execution và workflow"](/ch5#sec_encoding_dataflow_workflows)). Cách này gọi là *sao chép máy trạng thái* (*state machine replication*), và ta sẽ bàn lại ở [Chương 10](/ch10#ch_consistency).

#### Sharding {#sharding}

Thực thi mọi giao dịch tuần tự giúp kiểm soát đồng thời đơn giản hơn nhiều, nhưng cũng giới hạn thông lượng giao dịch của cơ sở dữ liệu trong khả năng xử lý của một máy, một lõi CPU. Giao dịch chỉ đọc có thể chạy nơi khác nhờ snapshot isolation, nhưng với ứng dụng có thông lượng ghi cao, bộ xử lý giao dịch đơn luồng có thể thành nút thắt nghiêm trọng.

Để mở rộng ra nhiều lõi CPU và nhiều node, có thể shard dữ liệu như VoltDB (xem [Chương 7](/ch7#ch_sharding)). Nếu mỗi giao dịch chỉ đọc ghi dữ liệu trong một shard duy nhất, mỗi shard có thể có luồng xử lý giao dịch riêng, chạy độc lập với nhau. Khi đó có thể gán mỗi lõi CPU một shard để thông lượng giao dịch tăng tuyến tính theo số lõi[^59].

Nhưng hễ giao dịch cần truy cập nhiều shard, cơ sở dữ liệu phải điều phối giữa tất cả shard liên quan. Thủ tục lưu trữ phải chạy đồng bộ nhịp trên các shard đó để bảo đảm tính serializable của toàn hệ thống.

Giao dịch xuyên shard có thêm chi phí điều phối nên chậm hơn nhiều so với giao dịch một shard. VoltDB báo cáo thông lượng ghi xuyên shard khoảng 1.000 lần mỗi giây, thấp hơn thông lượng một shard vài bậc độ lớn, và không thể tăng bằng cách thêm máy[^61]. Nghiên cứu gần đây đã tìm cách làm giao dịch nhiều shard mở rộng tốt hơn[^63].

Giao dịch có giới hạn được trong một shard hay không phụ thuộc rất nhiều vào cấu trúc dữ liệu mà ứng dụng dùng. Dữ liệu khóa-giá trị đơn giản thường dễ shard, còn dữ liệu có nhiều chỉ mục phụ (*secondary index*) thì rất có thể cần điều phối xuyên shard nhiều (xem ["Sharding và chỉ mục phụ (*secondary index*)"](/ch7#sec_sharding_secondary_indexes)).

#### Tóm tắt về thực thi tuần tự {#summary-of-serial-execution}

Với những ràng buộc nhất định, thực thi giao dịch tuần tự đã trở thành cách khả thi để hiện thực cô lập serializable:

* Mỗi giao dịch phải ngắn và nhanh, vì một giao dịch chậm là đủ kéo lùi toàn bộ việc xử lý giao dịch.
* Cách này phù hợp nhất khi tập dữ liệu đang hoạt động nằm vừa trong bộ nhớ. Dữ liệu ít truy cập có thể chuyển xuống đĩa, nhưng nếu giao dịch đơn luồng cần truy cập chúng thì hệ thống sẽ rất chậm.
* Thông lượng ghi phải đủ thấp để một lõi CPU xử lý được, nếu không, giao dịch phải có khả năng chia shard mà không cần điều phối xuyên shard.
* Giao dịch xuyên shard có thể làm được, nhưng thông lượng của chúng rất khó mở rộng.

### Khóa hai pha (2PL) {#sec_transactions_2pl}

Trong khoảng 30 năm, lĩnh vực cơ sở dữ liệu chỉ có một thuật toán serializable được dùng rộng rãi: *khóa hai pha* (*two-phase locking*, 2PL). Đôi khi nó còn được gọi là *khóa hai pha nghiêm ngặt mạnh* (*strong strict two-phase locking*, SS2PL) để phân biệt với các biến thể khác của 2PL.


> [!TIP] 2PL không phải 2PC
>
> Khóa hai pha (2PL) và commit hai pha (2PC) là hai thứ hoàn toàn khác nhau. 2PL cung cấp cô lập serializable, còn 2PC cung cấp commit nguyên tử (*atomic commitment*) cho cơ sở dữ liệu phân tán (xem ["Commit hai pha (2PC)"](/ch8#sec_transactions_2pc)). Để khỏi nhầm lẫn, tốt nhất hãy coi chúng là hai khái niệm riêng biệt và bỏ qua sự trùng tên đáng tiếc này.

Ở trên ta đã thấy khóa thường dùng để ngăn dirty write (xem ["Không có dirty write"](/ch8#sec_transactions_dirty_write)): nếu hai giao dịch cùng ghi một đối tượng, khóa buộc bên ghi thứ hai phải đợi đến khi giao dịch thứ nhất commit hoặc abort mới được tiếp tục.

Khóa hai pha tương tự, nhưng yêu cầu khóa chặt chẽ hơn nhiều. Chừng nào chưa có thao tác ghi, nhiều giao dịch có thể đọc cùng một đối tượng đồng thời, nhưng hễ có giao dịch muốn ghi (sửa hoặc xóa) đối tượng, nó phải có quyền truy cập độc quyền:

* Nếu giao dịch A đã đọc một đối tượng và giao dịch B muốn ghi đối tượng đó, B phải đợi A commit hoặc abort mới tiếp tục được. (Điều này bảo đảm B không thể đổi đối tượng mà A không hay biết.)
* Nếu giao dịch A đã ghi một đối tượng và giao dịch B muốn đọc đối tượng đó, B phải đợi A commit hoặc abort mới tiếp tục được. (2PL không cho phép đọc phiên bản cũ của đối tượng như trong {{< xref fig="8-4" page="/ch8" anchor="fig_transactions_read_committed" >}}Hình 8-4{{< /xref >}}.)

Trong 2PL, ghi chặn ghi khác, cũng chặn đọc; ngược lại, đọc cũng chặn ghi. Còn snapshot isolation theo nguyên tắc "*đọc không chặn ghi, ghi không chặn đọc*" (xem ["Kiểm soát đồng thời đa phiên bản (MVCC)"](/ch8#sec_transactions_snapshot_impl)), đó là khác biệt then chốt giữa snapshot isolation và khóa hai pha. Mặt khác, 2PL cung cấp serializable nên ngăn được mọi race condition đã bàn ở trên, gồm cả lost update và write skew.

#### Hiện thực hóa khóa hai pha {#implementation-of-two-phase-locking}

Mức cô lập serializable của MySQL (InnoDB) và SQL Server, cũng như mức repeatable read của Db2, đều dùng 2PL[^29].

Cơ sở dữ liệu chặn thao tác đọc và ghi bằng cách đặt một khóa cho mỗi đối tượng. Khóa có thể ở *chế độ chia sẻ* (*shared mode*) hoặc *chế độ độc quyền* (*exclusive mode*), còn gọi là khóa *multi-reader single-writer* (nhiều người đọc, một người ghi). Quy tắc như sau:

* Muốn đọc một đối tượng, giao dịch phải lấy khóa ở chế độ chia sẻ trước. Nhiều giao dịch có thể cùng giữ khóa chia sẻ, nhưng nếu giao dịch khác đã giữ khóa độc quyền trên đối tượng, giao dịch đọc phải đợi.
* Muốn ghi một đối tượng, giao dịch phải lấy khóa ở chế độ độc quyền trước. Khi đó không cho giao dịch nào khác giữ khóa ở bất kỳ chế độ nào, nếu đối tượng đã có khóa, giao dịch ghi phải đợi.
* Nếu giao dịch đọc rồi ghi, nó có thể nâng khóa chia sẻ lên khóa độc quyền. Quy tắc nâng khóa giống như lấy thẳng khóa độc quyền.
* Sau khi lấy được khóa, giao dịch phải giữ đến hết giao dịch (commit hoặc abort). Đây chính là nguồn gốc của tên "hai pha": pha thứ nhất lấy khóa trong lúc giao dịch thực thi, pha thứ hai thả toàn bộ khóa một lượt khi giao dịch kết thúc.

Vì dùng rất nhiều khóa, rất dễ xảy ra tình huống giao dịch A đợi giao dịch B nhả khóa, trong khi B lại đợi A, gọi là *deadlock*. Cơ sở dữ liệu tự động phát hiện deadlock giữa các giao dịch và abort một giao dịch để các giao dịch khác tiếp tục tiến triển. Giao dịch bị abort sẽ được ứng dụng thử lại.

#### Hiệu năng của khóa hai pha {#performance-of-two-phase-locking}

Nhược điểm lớn nhất của khóa hai pha, và cũng là lý do từ thập niên 1970 nó không được dùng ở mọi cơ sở dữ liệu, là hiệu năng: dưới khóa hai pha, thông lượng giao dịch và thời gian phản hồi truy vấn đều kém đáng kể so với cô lập yếu.

Một phần là do chi phí lấy và nhả rất nhiều khóa, nhưng quan trọng hơn là do mức đồng thời giảm. Theo thiết kế, hễ hai giao dịch đồng thời thực hiện thao tác nào có khả năng gây race condition, một trong hai phải đợi giao dịch kia xong.

Ví dụ, nếu một giao dịch cần đọc cả bảng để backup, truy vấn phân tích hay kiểm tra tính toàn vẹn (xem ["Snapshot isolation và repeatable read"](/ch8#sec_transactions_snapshot_isolation)), nó phải lấy khóa chia sẻ trên toàn bộ bảng. Giao dịch đọc trước hết phải đợi mọi giao dịch đang ghi bảng kết thúc. Sau đó, trong lúc nó đọc cả bảng (bảng lớn có thể mất rất lâu), mọi giao dịch muốn ghi vào bảng đều bị chặn cho đến khi giao dịch chỉ đọc lớn này commit. Thực tế là cơ sở dữ liệu không nhận được thao tác ghi nào trong một thời gian rất dài.

Vì vậy, độ trễ của cơ sở dữ liệu dùng 2PL có thể rất thất thường, nếu khối lượng công việc có tranh chấp, độ trễ ở phân vị cao có thể cực kỳ tệ (xem ["Mô tả hiệu năng"](/ch2#sec_introduction_percentiles)). Chỉ một giao dịch chậm, hoặc một giao dịch truy cập nhiều dữ liệu và lấy nhiều khóa, cũng có thể khiến phần còn lại của hệ thống đứng khựng.

Deadlock cũng xảy ra với read committed dựa trên khóa, nhưng với serializable kiểu 2PL thì deadlock thường xuyên hơn (tùy mẫu truy cập của giao dịch). Điều này gây thêm vấn đề hiệu năng: khi giao dịch bị abort vì deadlock và được thử lại, nó phải làm lại toàn bộ từ đầu. Nếu deadlock xảy ra thường xuyên, lượng tính toán lãng phí có thể rất đáng kể.

#### Khóa vị từ {#predicate-locks}

Phần mô tả khóa ở trên bỏ qua một chi tiết tinh tế nhưng quan trọng. ["Phantom gây ra write skew"](/ch8#sec_transactions_phantom) đã giới thiệu *phantom*: một giao dịch làm thay đổi kết quả truy vấn tìm kiếm của giao dịch khác. Cơ sở dữ liệu cung cấp cô lập serializable phải ngăn được phantom.

Trong ví dụ đặt phòng họp, nếu một giao dịch đã tìm các lịch đặt hiện có của một phòng trong một khung giờ (xem {{< xref eg="8-2" page="/ch8" anchor="fig_transactions_meeting_rooms" >}}Ví dụ 8-2{{< /xref >}}), thì giao dịch khác không được chèn hoặc cập nhật lịch đặt của cùng phòng, cùng khung giờ đồng thời. (Đặt đồng thời các phòng khác, hoặc cùng phòng nhưng khung giờ khác không ảnh hưởng thì không sao.)

Làm sao hiện thực điều này? Về mặt khái niệm, cần dùng *khóa vị từ* (*predicate lock*)[^4]. Nó hoạt động giống khóa chia sẻ và khóa độc quyền đã nói ở trên, nhưng không thuộc về một đối tượng cụ thể (như một dòng trong bảng), mà thuộc về mọi đối tượng thỏa một điều kiện tìm kiếm, chẳng hạn:

```
SELECT * FROM bookings
 WHERE room_id = 123 AND
 end_time > '2025-01-01 12:00' AND
 start_time < '2025-01-01 13:00';
```

Khóa vị từ hạn chế truy cập như sau:

* Nếu giao dịch A muốn đọc các đối tượng thỏa một điều kiện (như `SELECT` ở trên), nó phải lấy khóa vị từ ở chế độ chia sẻ trên điều kiện truy vấn. Nếu giao dịch B đang giữ khóa độc quyền trên bất kỳ đối tượng nào thỏa điều kiện, A phải đợi B nhả khóa mới truy vấn được.
* Nếu giao dịch A muốn chèn, cập nhật hoặc xóa đối tượng, trước hết nó phải kiểm tra giá trị cũ hoặc mới có thỏa khóa vị từ nào đang có không. Nếu thấy giao dịch B giữ khóa vị từ khớp, A phải đợi B commit hoặc abort mới tiếp tục.

Điểm mấu chốt là khóa vị từ áp dụng cả với các đối tượng chưa tồn tại trong cơ sở dữ liệu nhưng có thể được thêm vào sau này (tức phantom). Nếu khóa hai pha bao gồm khóa vị từ, cơ sở dữ liệu ngăn được mọi dạng write skew và race condition khác, đưa cô lập lên mức serializable.

#### Khóa phạm vi chỉ mục (*index*) {#sec_transactions_2pl_range}

Đáng tiếc, khóa vị từ hoạt động không tốt: khi các giao dịch đang hoạt động giữ nhiều khóa, việc kiểm tra có khóa nào khớp rất tốn thời gian. Vì vậy, hầu hết cơ sở dữ liệu dùng 2PL thực tế hiện thực *khóa phạm vi chỉ mục (*index*)* (*index-range lock*, còn gọi là *next-key locking*), một dạng xấp xỉ đơn giản hóa của khóa vị từ[^54] [^64].

Nới lỏng vị từ để khớp một tập đối tượng lớn hơn là một cách đơn giản hóa an toàn. Ví dụ, muốn khóa các lịch đặt phòng 123 từ 12 giờ trưa đến 1 giờ chiều, có thể xấp xỉ thành khóa mọi lịch đặt của phòng 123 cả ngày, hoặc xấp xỉ thành khóa lịch đặt của mọi phòng từ 12 giờ trưa đến 1 giờ chiều. Làm vậy an toàn, vì mọi thao tác ghi thỏa vị từ gốc đương nhiên cũng thỏa điều kiện xấp xỉ.

Cơ sở dữ liệu đặt phòng họp rất có thể có chỉ mục (*index*) trên cột `room_id`, hoặc chỉ mục (*index*) trên `start_time` và `end_time` (nếu không, truy vấn trên sẽ rất chậm với cơ sở dữ liệu lớn):

* Giả sử chỉ mục (*index*) nằm trên `room_id`, và cơ sở dữ liệu dùng nó để tìm các lịch đặt hiện có của phòng 123. Cơ sở dữ liệu có thể gắn khóa chia sẻ ngay vào mục chỉ mục (*index*) này, biểu thị rằng một giao dịch đã tìm lịch đặt của phòng 123.
* Nếu cơ sở dữ liệu dùng chỉ mục (*index*) thời gian để tìm lịch đặt hiện có, nó có thể gắn khóa chia sẻ vào một dải giá trị trong chỉ mục (*index*), biểu thị rằng một giao dịch đã tìm các lịch đặt chồng lên khoảng từ 12 giờ trưa đến 1 giờ chiều ngày 1 tháng 1 năm 2025.

Dù dùng chỉ mục (*index*) nào, phần xấp xỉ của điều kiện tìm kiếm cũng bám vào một trong các chỉ mục (*index*). Giao dịch khác muốn chèn, cập nhật hoặc xóa lịch đặt của cùng phòng và/hoặc khoảng thời gian chồng chéo phải cập nhật cùng phần đó của chỉ mục (*index*), nên sẽ đụng vào khóa chia sẻ và buộc phải đợi đến khi khóa được nhả.

Như vậy ngăn được phantom và write skew một cách hiệu quả. Khóa phạm vi chỉ mục (*index*) không chính xác bằng khóa vị từ, có thể khóa phạm vi đối tượng lớn hơn mức thật sự cần để giữ serializable, nhưng chi phí thấp hơn nhiều, nên là một sự thỏa hiệp tốt.

Nếu không có chỉ mục (*index*) phù hợp để gắn khóa phạm vi, cơ sở dữ liệu có thể lùi về khóa chia sẻ trên cả bảng. Cách này chặn mọi giao dịch khác ghi vào bảng nên hiệu năng rất kém, nhưng là phương án dự phòng an toàn.

### Serializable snapshot isolation (SSI) {#sec_transactions_ssi}

Đến đây, bức tranh kiểm soát đồng thời trong cơ sở dữ liệu khá ảm đạm: một bên là khóa hai pha hiệu năng kém và thực thi tuần tự khó mở rộng, bên kia là cô lập yếu có hiệu năng tốt nhưng dễ dính race condition như lost update, write skew và phantom. Phải chăng cô lập serializable và hiệu năng tốt vốn không thể đi đôi?

Có vẻ không phải vậy. Một thuật toán tên *serializable snapshot isolation* (SSI) cung cấp đầy đủ tính serializable mà chỉ tốn thêm chút hiệu năng so với snapshot isolation. SSI tương đối mới, được đề xuất lần đầu vào năm 2008[^53] [^65].

Ngày nay, SSI và các thuật toán tương tự được dùng trong cơ sở dữ liệu đơn node (mức serializable của PostgreSQL[^54], OLTP trong bộ nhớ/Hekaton của SQL Server[^66] và HyPer[^67]), cơ sở dữ liệu phân tán (CockroachDB[^5] và FoundationDB[^8]), cũng như các storage engine nhúng như BadgerDB.

#### Kiểm soát đồng thời bi quan và lạc quan {#pessimistic-versus-optimistic-concurrency-control}

Khóa hai pha thuộc loại cơ chế kiểm soát đồng thời *bi quan* (*pessimistic*). Nguyên tắc của nó là: hễ có khả năng sai (ví dụ giao dịch khác đang giữ khóa) thì đợi, đến khi mọi thứ an toàn trở lại mới hành động. Điều này giống *loại trừ lẫn nhau* (*mutual exclusion*) dùng để bảo vệ cấu trúc dữ liệu trong lập trình đa luồng.

Theo một nghĩa nào đó, thực thi tuần tự đẩy sự bi quan lên cực điểm: về bản chất, mỗi giao dịch trong lúc thực thi đều như đang giữ khóa độc quyền trên cả cơ sở dữ liệu (hoặc một shard của nó). Bù lại, mỗi giao dịch phải chạy thật nhanh để "khóa" đó chỉ bị giữ rất ngắn.

Ngược lại, serializable snapshot isolation là kỹ thuật kiểm soát đồng thời *lạc quan* (*optimistic*). "Lạc quan" ở đây nghĩa là: dù có nguy cơ, giao dịch không bị chặn mà cứ tiếp tục chạy, với hy vọng cuối cùng không có vấn đề gì. Đến khi giao dịch sắp commit, cơ sở dữ liệu mới kiểm tra xem có chuyện xấu xảy ra không (tức cô lập có bị phá vỡ không), nếu có thì giao dịch bị abort và thử lại. Chỉ những giao dịch có kết quả thực thi serializable mới được commit.

Kiểm soát đồng thời lạc quan là ý tưởng cũ[^68], và ưu nhược điểm của nó cũng đã được tranh luận nhiều năm[^69]. Khi tranh chấp nghiêm trọng (nhiều giao dịch truy cập cùng đối tượng), nó hoạt động kém, vì một tỷ lệ lớn giao dịch buộc phải abort. Nếu hệ thống đã gần đạt thông lượng tối đa, tải phụ do thử lại giao dịch còn kéo hiệu năng xuống thêm.

Tuy nhiên, nếu hệ thống còn nhiều dư địa và tranh chấp giữa các giao dịch không nặng, kiểm soát đồng thời lạc quan thường tốt hơn phương án bi quan. Thao tác nguyên tử giao hoán cũng giúp giảm tranh chấp: ví dụ, khi nhiều giao dịch cùng tăng một bộ đếm, thứ tự áp dụng các lần tăng không quan trọng (miễn giao dịch đó không đọc bộ đếm), nên mọi lần tăng đồng thời đều thực hiện được mà không xung đột.

Đúng như tên gọi, SSI dựa trên snapshot isolation: mọi thao tác đọc trong giao dịch đều từ một snapshot nhất quán của cơ sở dữ liệu (xem ["Snapshot isolation và repeatable read"](/ch8#sec_transactions_snapshot_isolation)). Trên nền đó, SSI thêm thuật toán phát hiện xung đột serializable giữa đọc và ghi, và quyết định giao dịch nào cần abort.

#### Quyết định dựa trên tiền đề lỗi thời {#decisions-based-on-an-outdated-premise}

Khi bàn write skew trong snapshot isolation (xem ["Write skew và phantom"](/ch8#sec_transactions_write_skew)), ta thấy lặp đi lặp lại một mẫu: giao dịch đọc dữ liệu từ cơ sở dữ liệu, kiểm tra kết quả truy vấn, rồi tùy kết quả mà thực hiện một thao tác (ghi vào cơ sở dữ liệu). Tuy nhiên, dưới snapshot isolation, đến lúc giao dịch commit thì kết quả truy vấn ban đầu có thể đã lỗi thời, vì dữ liệu có thể đã bị sửa trong lúc đó.

Nói cách khác, giao dịch hành động dựa trên một *tiền đề* — một sự thật đúng lúc giao dịch bắt đầu, chẳng hạn "hiện có hai bác sĩ đang trực". Khi giao dịch sắp commit, dữ liệu gốc có thể đã đổi, tiền đề ban đầu không còn đúng.

Khi ứng dụng gửi truy vấn (ví dụ "hiện có bao nhiêu bác sĩ đang trực?"), cơ sở dữ liệu không biết logic ứng dụng sẽ dùng kết quả truy vấn thế nào. Để an toàn, cơ sở dữ liệu phải giả định rằng hễ kết quả truy vấn (tức tiền đề) thay đổi thì các thao tác ghi trong giao dịch đó có thể không còn hợp lệ. Nói cách khác, giữa truy vấn và thao tác ghi trong giao dịch có thể có phụ thuộc nhân quả. Để cung cấp cô lập serializable, cơ sở dữ liệu phải phát hiện được trường hợp giao dịch có thể đã hành động dựa trên tiền đề lỗi thời, và abort giao dịch trong trường hợp đó.

Làm sao cơ sở dữ liệu biết kết quả truy vấn có thể đã đổi? Có hai trường hợp cần xét:

* Phát hiện đọc phải phiên bản đối tượng MVCC lỗi thời (thao tác ghi chưa commit xảy ra trước lúc đọc);
* Phát hiện thao tác ghi ảnh hưởng đến lần đọc trước đó (thao tác ghi xảy ra sau lúc đọc).

#### Phát hiện đọc MVCC cũ {#detecting-stale-mvcc-reads}

Nhắc lại, snapshot isolation thường được hiện thực bằng kiểm soát đồng thời đa phiên bản (MVCC; xem ["Kiểm soát đồng thời đa phiên bản (MVCC)"](/ch8#sec_transactions_snapshot_impl)). Khi giao dịch đọc từ một snapshot nhất quán của cơ sở dữ liệu MVCC, nó bỏ qua mọi thao tác ghi mà giao dịch khác chưa commit tại lúc tạo snapshot.

Trong {{< xref fig="8-10" page="/ch8" anchor="fig_transactions_detect_mvcc" >}}Hình 8-10{{< /xref >}}, giao dịch 43 thấy `on_call = true` của Aaliyah, vì giao dịch 42 sửa trạng thái trực ca của cô chưa commit. Nhưng khi giao dịch 43 sắp commit thì giao dịch 42 đã commit. Nghĩa là thao tác ghi bị bỏ qua khi đọc snapshot nhất quán nay đã có hiệu lực, và tiền đề của giao dịch 43 không còn đúng. Nếu bên ghi chèn dữ liệu chưa từng tồn tại, tình huống còn phức tạp hơn (xem ["Phantom gây ra write skew"](/ch8#sec_transactions_phantom)). ["Phát hiện thao tác ghi ảnh hưởng đến lần đọc trước"](/ch8#sec_detecting_writes_affect_reads) sẽ giới thiệu cách SSI phát hiện phantom write.

{{< fig num="8-10" id="fig_transactions_detect_mvcc" src="/fig/ddia_0810.png" caption="Phát hiện khi giao dịch đọc giá trị lỗi thời từ snapshot MVCC." class="ddia-figure ddia-figure--standard" width="2658" height="1613" />}}


Để ngăn bất thường này, cơ sở dữ liệu phải theo dõi khi nào giao dịch bỏ qua thao tác ghi của giao dịch khác do quy tắc hiển thị MVCC. Khi giao dịch sắp commit, nó kiểm tra xem các thao tác ghi bị bỏ qua đó đã commit chưa, nếu rồi thì giao dịch phải bị abort.

Vì sao phải đợi đến lúc commit? Sao không abort giao dịch 43 ngay khi phát hiện đọc lỗi thời? Nếu giao dịch 43 chỉ đọc thì không có nguy cơ write skew nên không cần abort, mà lúc nó đọc, cơ sở dữ liệu cũng chưa biết sau đó nó có ghi hay không. Hơn nữa, giao dịch 42 có thể cuối cùng bị abort, hoặc vẫn chưa commit khi giao dịch 43 commit, khi đó lần đọc rốt cuộc không hề lỗi thời. Nhờ tránh các lần abort không cần thiết, SSI giữ được ưu điểm của snapshot isolation là hỗ trợ đọc snapshot nhất quán trong thời gian dài.

#### Phát hiện thao tác ghi ảnh hưởng đến lần đọc trước {#sec_detecting_writes_affect_reads}

Trường hợp thứ hai là dữ liệu bị đọc rồi sau đó bị giao dịch khác sửa, như {{< xref fig="8-11" page="/ch8" anchor="fig_transactions_detect_index_range" >}}Hình 8-11{{< /xref >}}.

{{< fig num="8-11" id="fig_transactions_detect_index_range" src="/fig/ddia_0811.png" caption="Trong serializable snapshot isolation, phát hiện khi một giao dịch sửa dữ liệu mà giao dịch khác đã đọc." class="ddia-figure ddia-figure--standard" width="2658" height="1654" />}}


Khi bàn khóa hai pha, ta đã giới thiệu khóa phạm vi chỉ mục (*index*) (xem ["Khóa phạm vi chỉ mục (*index*)"](/ch8#sec_transactions_2pl_range)), nhờ đó cơ sở dữ liệu khóa được mọi dòng thỏa một điều kiện truy vấn, ví dụ `WHERE shift_id = 1234`. SSI dùng kỹ thuật tương tự, khác ở chỗ khóa của SSI không chặn giao dịch khác.

Trong {{< xref fig="8-11" page="/ch8" anchor="fig_transactions_detect_index_range" >}}Hình 8-11{{< /xref >}}, các giao dịch 42 và 43 đều truy vấn bác sĩ trực ở ca `1234`. Nếu có chỉ mục (*index*) trên `shift_id`, cơ sở dữ liệu dùng mục chỉ mục (*index*) 1234 để ghi lại rằng giao dịch 42 và 43 đã đọc dữ liệu này. (Nếu không có chỉ mục (*index*), có thể theo dõi ở mức bảng.) Thông tin này chỉ cần giữ một thời gian: khi giao dịch kết thúc (commit hoặc abort) và mọi giao dịch đồng thời với nó cũng kết thúc, cơ sở dữ liệu có thể quên nó đã đọc dữ liệu nào.

Khi giao dịch ghi vào cơ sở dữ liệu, nó phải tra trong chỉ mục (*index*) để tìm các giao dịch khác vừa đọc dữ liệu bị ảnh hưởng. Quá trình này giống lấy khóa ghi trên phạm vi khóa bị ảnh hưởng, nhưng nó không chặn đến khi giao dịch đọc commit, thay vào đó nó như một dây cảnh giới: chỉ thông báo cho các giao dịch liên quan rằng dữ liệu chúng đọc có thể đã lỗi thời.

Trong {{< xref fig="8-11" page="/ch8" anchor="fig_transactions_detect_index_range" >}}Hình 8-11{{< /xref >}}, giao dịch 43 thông báo cho giao dịch 42 rằng dữ liệu nó đọc trước đó đã lỗi thời, còn giao dịch 42 cũng thông báo ngược lại cho giao dịch 43. Giao dịch 42 commit trước và thành công: dù thao tác ghi của giao dịch 43 ảnh hưởng đến 42, giao dịch 43 chưa commit nên thao tác ghi chưa có hiệu lực. Đến khi giao dịch 43 sắp commit, thao tác ghi xung đột của giao dịch 42 đã commit, nên giao dịch 43 phải abort.

#### Hiệu năng của serializable snapshot isolation {#performance-of-serializable-snapshot-isolation}

Như mọi khi, nhiều chi tiết kỹ thuật ảnh hưởng đến hiệu quả thực tế của thuật toán. Chẳng hạn, phải cân nhắc mức chi tiết khi theo dõi đọc/ghi của giao dịch. Nếu cơ sở dữ liệu ghi lại chi tiết hoạt động của từng giao dịch, nó xác định chính xác giao dịch nào cần abort, nhưng chi phí ghi sổ có thể rất lớn, còn theo dõi thô hơn thì nhanh hơn, nhưng có thể abort cả những giao dịch lẽ ra không cần.

Có trường hợp giao dịch đọc phải thông tin bị giao dịch khác ghi đè cũng không sao: kết hợp với các sự kiện khác, đôi khi có thể chứng minh kết quả thực thi vẫn serializable. PostgreSQL dùng lý thuyết này để giảm số lần abort không cần thiết[^14] [^54].

So với khóa hai pha, ưu điểm chính của serializable snapshot isolation là giao dịch không phải bị chặn để đợi khóa do giao dịch khác giữ. Như snapshot isolation, ghi không chặn đọc và đọc không chặn ghi. Nguyên tắc thiết kế này làm độ trễ truy vấn dễ dự đoán và ít dao động hơn. Đặc biệt, truy vấn chỉ đọc có thể chạy trên snapshot nhất quán mà không cần khóa nào, rất hấp dẫn với khối lượng công việc thiên về đọc.

So với thực thi tuần tự, serializable snapshot isolation không bị giới hạn bởi thông lượng của một lõi CPU. Ví dụ, FoundationDB phân tán việc phát hiện xung đột serializable ra nhiều máy nên có thể mở rộng đến thông lượng rất cao. Dù dữ liệu được shard trên nhiều máy, giao dịch vẫn đọc ghi được nhiều shard mà vẫn bảo đảm cô lập serializable.

So với snapshot isolation không serializable, việc kiểm tra vi phạm serializable gây ra một ít chi phí hiệu năng. Chi phí này lớn đến đâu vẫn còn tranh cãi: có người cho rằng kiểm tra serializable không đáng[^70]; người khác cho rằng ngày nay hiệu năng serializable đã đủ tốt, không cần dùng snapshot isolation yếu hơn nữa[^67].

Tỷ lệ abort ảnh hưởng đáng kể đến hiệu năng tổng thể của SSI. Giao dịch đọc ghi dữ liệu trong thời gian dài rất dễ gặp xung đột và bị abort, nên SSI đòi hỏi giao dịch đọc–ghi phải khá ngắn (giao dịch chỉ đọc chạy lâu thì không sao). Dù vậy, so với khóa hai pha hay thực thi tuần tự, SSI ít nhạy cảm với giao dịch chậm hơn.

## Giao dịch phân tán {#sec_transactions_distributed}

Các mục trước tập trung vào kiểm soát đồng thời cần thiết để hiện thực tính cô lập (chữ I trong ACID). Các thuật toán đã giới thiệu áp dụng được cho cả cơ sở dữ liệu đơn node lẫn phân tán. Dù thuật toán kiểm soát đồng thời gặp thách thức khi mở rộng (ví dụ phải kiểm tra serializable phân tán cho SSI), ý tưởng chung của kiểm soát đồng thời phân tán vẫn tương tự đơn node[^8].

Chuyển sang giao dịch phân tán, tính nhất quán và tính bền vững cũng không đổi nhiều, nhưng tính nguyên tử cần đặc biệt cẩn thận.

Với giao dịch chỉ thực thi trên một node cơ sở dữ liệu, tính nguyên tử thường do storage engine hiện thực. Khi client yêu cầu node cơ sở dữ liệu commit, cơ sở dữ liệu trước hết ghi bền vững giao dịch (thường vào write-ahead log; xem ["Làm cho B-tree đáng tin cậy"](/ch4#sec_storage_btree_wal)), rồi nối thêm một bản ghi commit vào log trên đĩa. Nếu cơ sở dữ liệu sập giữa chừng, khi node khởi động lại, giao dịch được khôi phục từ log: nếu bản ghi commit đã xuống đĩa thành công trước khi sập thì giao dịch coi như đã commit, nếu không thì mọi thao tác ghi của giao dịch bị rollback.

Vì vậy, trên một node, giao dịch có commit được hay không phụ thuộc then chốt vào *thứ tự* dữ liệu được ghi bền xuống đĩa: ghi dữ liệu trước, rồi mới ghi bản ghi commit[^22]. Thời điểm quyết định giao dịch commit hay abort là lúc đĩa hoàn tất việc ghi bản ghi commit: trước đó giao dịch vẫn có thể bị abort do sập; sau đó, dù cơ sở dữ liệu sập, giao dịch đã commit. Nói cách khác, chính một thiết bị duy nhất — bộ điều khiển của một ổ đĩa gắn với một node cụ thể — làm cho commit có tính nguyên tử.

Nhưng nếu giao dịch liên quan đến nhiều node thì sao? Ví dụ, cơ sở dữ liệu đã shard có thể có giao dịch đa đối tượng, hoặc có chỉ mục phụ (*secondary index*) toàn cục mà mục chỉ mục (*index*) và dữ liệu chính nằm trên các node khác nhau (xem ["Sharding và chỉ mục phụ (*secondary index*)"](/ch7#sec_sharding_secondary_indexes)). Hầu hết kho dữ liệu phân tán "NoSQL" không hỗ trợ loại giao dịch phân tán này, nhưng nhiều cơ sở dữ liệu quan hệ phân tán thì có.

Lúc này, chỉ gửi yêu cầu commit đến mọi node và để mỗi node commit giao dịch độc lập là chưa đủ. Như {{< xref fig="8-12" page="/ch8" anchor="fig_transactions_non_atomic" >}}Hình 8-12{{< /xref >}} cho thấy, rất dễ xảy ra commit thành công trên một số node nhưng thất bại trên số khác:

* Một số node có thể phát hiện vi phạm ràng buộc hoặc xung đột nên buộc phải abort, trong khi các node khác commit thành công;
* Một số yêu cầu commit có thể thất lạc trên mạng, cuối cùng bị abort do quá hạn, trong khi các yêu cầu khác đến nơi thuận lợi;
* Một số node có thể sập trước khi bản ghi commit được ghi đầy đủ, nên rollback giao dịch khi khôi phục, trong khi các node khác commit thành công.

{{< fig num="8-12" id="fig_transactions_non_atomic" src="/fig/ddia_0812.png" caption="Khi giao dịch liên quan đến nhiều node cơ sở dữ liệu, nó có thể commit trên một số node và thất bại trên các node khác." class="ddia-figure ddia-figure--wide" width="2658" height="998" />}}


Nếu một số node commit còn số khác abort, các node sẽ không nhất quán với nhau. Và một khi giao dịch đã commit trên một node, dù sau đó phát hiện nó abort trên node khác, cũng không thể rút lại được. Vì dữ liệu sau khi commit sẽ hiển thị với các giao dịch khác ở mức *read committed* hoặc cô lập mạnh hơn. Chẳng hạn trong {{< xref fig="8-12" page="/ch8" anchor="fig_transactions_non_atomic" >}}Hình 8-12{{< /xref >}}, khi người dùng 1 phát hiện commit trên cơ sở dữ liệu 1 thất bại thì người dùng 2 đã đọc được dữ liệu do cùng giao dịch ghi trên cơ sở dữ liệu 2. Nếu sau đó mới abort giao dịch của người dùng 1, giao dịch của người dùng 2 cũng phải bị hoàn tác, vì dữ liệu nó dựa vào bị tuyên bố hồi tố là chưa từng tồn tại.

Cách tốt hơn là bảo đảm các node tham gia giao dịch hoặc tất cả cùng commit, hoặc tất cả cùng abort, tuyệt đối không để hai kết quả lẫn lộn. Đó chính là bài toán *commit nguyên tử* (*atomic commitment*).

### Commit hai pha (2PC) {#sec_transactions_2pc}

Commit hai pha là thuật toán để commit giao dịch nguyên tử trên nhiều node, và là thuật toán kinh điển trong cơ sở dữ liệu phân tán[^13] [^71] [^72]. Một số cơ sở dữ liệu dùng 2PC nội bộ, và nó cũng được mở cho ứng dụng dưới dạng *giao dịch XA*[^73] (ví dụ Java Transaction API hỗ trợ XA), hoặc qua WS-AtomicTransaction cho dịch vụ web SOAP[^74] [^75].

Quy trình cơ bản của 2PC như {{< xref fig="8-13" page="/ch8" anchor="fig_transactions_two_phase_commit" >}}Hình 8-13{{< /xref >}}. Giao dịch đơn node chỉ cần một yêu cầu commit, còn 2PC chia quá trình commit hoặc abort thành hai pha, tên gọi bắt nguồn từ đó.

{{< fig num="8-13" id="fig_transactions_two_phase_commit" src="/fig/ddia_0813.png" caption="Một lần thực thi thành công của commit hai pha (2PC)." class="ddia-figure ddia-figure--wide" width="2880" height="969" />}}


2PC đưa vào một thành phần mà giao dịch đơn node thường không có: *coordinator* (còn gọi là *transaction manager*, trình quản lý giao dịch). Coordinator thường được hiện thực dưới dạng thư viện chạy cùng tiến trình với ứng dụng khởi tạo giao dịch (ví dụ nhúng trong container Java EE), hoặc cũng có thể chạy thành tiến trình hoặc dịch vụ riêng. Narayana, JOTM, BTM và MSDTC đều là loại coordinator này.

Khi dùng 2PC, giao dịch phân tán bắt đầu như bình thường: ứng dụng đọc và ghi dữ liệu trên nhiều node cơ sở dữ liệu, các node này gọi là *participant* của giao dịch. Khi ứng dụng sẵn sàng commit, coordinator bước vào pha thứ nhất, gửi yêu cầu *prepare* đến từng participant để hỏi liệu nó có thể commit không, và thu thập phản hồi của các participant:

* Nếu mọi participant đều trả lời "có", tức đã sẵn sàng commit, coordinator ở pha thứ hai phát yêu cầu *commit* và thực sự thực hiện commit;
* Nếu bất kỳ participant nào trả lời "không", coordinator ở pha thứ hai gửi yêu cầu *abort* đến mọi node.

Quy trình này hơi giống lễ cưới truyền thống phương Tây: cha xứ lần lượt hỏi cô dâu và chú rể có đồng ý lấy nhau không, thường cả hai đều đáp "con đồng ý". Nhận được xác nhận của cả hai, cha xứ tuyên bố họ là vợ chồng — giao dịch commit, và tin vui được báo cho mọi khách mời. Nếu một trong hai không trả lời "đồng ý", buổi lễ bị hủy[^76].

#### Hệ thống của các lời hứa {#a-system-of-promises}

Chỉ nhìn mô tả ngắn ở trên, có thể chưa thấy vì sao commit hai pha bảo đảm được tính nguyên tử mà commit một pha trên nhiều node thì không. Rốt cuộc, trong quy trình hai pha, yêu cầu prepare và commit cũng có thể thất lạc. 2PC khác ở chỗ nào?

Để hiểu vì sao nó hiệu quả, cần phân tích quy trình kỹ hơn:

1. Khi ứng dụng muốn bắt đầu giao dịch phân tán, trước hết nó xin coordinator một ID giao dịch toàn cục duy nhất.
2. Ứng dụng bắt đầu một giao dịch đơn node trên mỗi participant và gắn ID giao dịch toàn cục vào đó. Mọi thao tác đọc và ghi đều thực hiện trong các giao dịch đơn node này. Nếu có vấn đề ở giai đoạn này (ví dụ node sập hoặc yêu cầu quá hạn), coordinator hoặc bất kỳ participant nào cũng có thể abort giao dịch.
3. Khi ứng dụng sẵn sàng commit, coordinator gửi yêu cầu prepare kèm ID giao dịch toàn cục đến mọi participant. Nếu bất kỳ yêu cầu nào thất bại hoặc quá hạn, coordinator gửi yêu cầu abort cho ID giao dịch đó đến mọi participant.
4. Khi participant nhận yêu cầu prepare, nó phải bảo đảm chắc chắn commit được giao dịch trong mọi hoàn cảnh. Điều này gồm việc ghi toàn bộ dữ liệu giao dịch xuống đĩa (sập, mất điện hay hết dung lượng đĩa đều không được là lý do để từ chối commit về sau), cũng như kiểm tra xung đột và vi phạm ràng buộc. Khi node trả lời "có" cho coordinator, nó đã hứa: chỉ cần được yêu cầu, giao dịch chắc chắn commit không lỗi. Nói cách khác, participant chưa thực sự commit nhưng đã từ bỏ quyền tự abort giao dịch.
5. Khi coordinator nhận đủ phản hồi cho mọi yêu cầu prepare, nó đưa ra quyết định cuối cùng là commit hay abort (chỉ commit khi mọi participant bỏ phiếu "có"). Coordinator phải ghi quyết định này vào log giao dịch trên đĩa, để dù sau đó có sập, khi khôi phục nó vẫn biết mình đã quyết định gì. Thời điểm này gọi là *điểm commit* (*commit point*).
6. Khi quyết định của coordinator đã xuống đĩa, nó gửi yêu cầu commit hoặc abort đến mọi participant. Nếu yêu cầu thất bại hoặc quá hạn, coordinator phải thử lại liên tục cho đến khi thành công. Lúc này không còn đường lui: nếu đã quyết định commit thì phải thực hiện đến cùng, dù cần thử lại bao nhiêu lần. Nếu participant sập trong thời gian này, khi khôi phục nó cũng phải commit giao dịch — đã bỏ phiếu "có" thì không được đổi ý.

Như vậy, giao thức có hai "điểm không thể quay lại" then chốt: khi participant bỏ phiếu "có", nó hứa sau này chắc chắn commit được (dù coordinator vẫn có thể quyết định abort); và khi coordinator đã quyết định, quyết định đó không thể rút lại. Chính các lời hứa này bảo đảm tính nguyên tử của 2PC. (Commit nguyên tử (*atomic commitment*) trên đơn node gộp hai việc này thành một bước: ghi bản ghi commit vào log giao dịch.)

Quay lại phép ẩn dụ đám cưới: trước khi nói "con đồng ý", bạn hay người bạn đời đều có thể hủy giao dịch bằng câu trả lời như "tuyệt đối không!", nhưng một khi đã nói "con đồng ý" thì không thể rút lại. Dù nói xong bạn ngất đi, không nghe cha xứ tuyên bố "giờ hai con là vợ chồng", cũng không thay đổi việc giao dịch đã commit. Khi tỉnh lại, bạn có thể hỏi cha xứ trạng thái của ID giao dịch toàn cục để biết mình đã kết hôn chưa, hoặc đợi cha xứ gửi lại yêu cầu commit — trong lúc bạn bất tỉnh, ông vẫn thử đi thử lại.

#### Coordinator hỏng {#coordinator-failure}

Ta đã bàn điều gì xảy ra khi participant hoặc mạng gặp sự cố trong 2PC: yêu cầu prepare nào thất bại hoặc quá hạn thì coordinator abort giao dịch; yêu cầu commit hoặc abort nào thất bại thì coordinator thử lại vô hạn. Nhưng khi chính coordinator sập thì không còn rõ ràng như vậy.

Nếu coordinator hỏng trước khi gửi yêu cầu prepare, participant có thể abort giao dịch an toàn. Nhưng khi participant đã nhận yêu cầu prepare và bỏ phiếu "có", nó không thể đơn phương abort nữa, mà phải đợi coordinator cho biết giao dịch commit hay abort. Nếu lúc này coordinator sập hoặc mạng lỗi, participant chỉ biết chờ. Giao dịch ở trạng thái này gọi là giao dịch *in doubt* hay *uncertain*.

{{< xref fig="8-14" page="/ch8" anchor="fig_transactions_2pc_crash" >}}Hình 8-14{{< /xref >}} minh họa tình huống này. Trong ví dụ, coordinator thực sự đã quyết định commit và cơ sở dữ liệu 2 đã nhận yêu cầu commit, nhưng coordinator sập trước khi kịp gửi yêu cầu commit cho cơ sở dữ liệu 1, nên cơ sở dữ liệu 1 không biết phải commit hay abort. Timeout cũng chẳng giúp ích: nếu cơ sở dữ liệu 1 tự abort sau khi quá hạn thì sẽ không nhất quán với cơ sở dữ liệu 2 đã commit, còn tự commit cũng không an toàn, vì participant kia có thể đã abort.

{{< fig num="8-14" id="fig_transactions_2pc_crash" src="/fig/ddia_0814.png" caption="Coordinator sập sau khi participant bỏ phiếu \"có\". Cơ sở dữ liệu 1 không biết nên commit hay abort." class="ddia-figure ddia-figure--wide" width="2880" height="1006" />}}


Không nhận được tin từ coordinator, participant không có cách nào biết nên commit hay abort. Về nguyên tắc, các participant có thể liên lạc với nhau để biết mỗi bên đã bỏ phiếu thế nào và đạt thỏa thuận nào đó, nhưng việc này không thuộc giao thức 2PC.

Cách duy nhất để hoàn tất 2PC là đợi coordinator khôi phục. Đó là lý do coordinator phải ghi quyết định commit hoặc abort vào log giao dịch trên đĩa trước khi gửi yêu cầu cho participant. Khi coordinator khôi phục, nó đọc log giao dịch để xác định trạng thái mọi giao dịch in doubt. Giao dịch nào không có bản ghi commit trong log của coordinator đều bị abort. Do đó, điểm commit của 2PC rốt cuộc là một lần commit nguyên tử (*atomic commitment*) đơn node bình thường trên coordinator.

#### Commit ba pha {#three-phase-commit}

Vì 2PC có thể bị kẹt đợi coordinator khôi phục, commit hai pha còn gọi là giao thức commit nguyên tử (*atomic commitment*) *chặn* (*blocking*). Về lý thuyết có thể thiết kế giao thức commit nguyên tử (*atomic commitment*) *không chặn* (*nonblocking*) để không bị kẹt khi node lỗi, nhưng làm được điều này trong thực tế không dễ.

Có người đề xuất *commit ba pha* (*three-phase commit*, 3PC) làm phương án thay cho 2PC[^13] [^77]. Tuy nhiên, 3PC giả định độ trễ mạng có giới hạn trên và thời gian phản hồi của node cũng có giới hạn trên. Hầu hết hệ thống thực tế có độ trễ mạng không giới hạn và tiến trình bị tạm dừng (xem [Chương 9](/ch9#ch_distributed)), nên 3PC không bảo đảm được tính nguyên tử.

Cách tốt hơn trong thực tế là thay coordinator đơn node bằng giao thức đồng thuận chịu lỗi. [Chương 10](/ch10#ch_consistency) sẽ giới thiệu cách làm.

### Giao dịch phân tán xuyên các hệ thống khác nhau {#sec_transactions_xa}

Giao dịch phân tán và commit hai pha có tiếng tăm lẫn lộn. Một mặt, chúng đem lại những bảo đảm an toàn quan trọng khó đạt bằng cách khác; mặt khác, chúng bị chỉ trích vì gây vấn đề vận hành, làm hại hiệu năng nghiêm trọng, và hứa những điều làm không nổi[^78] [^79] [^80] [^81]. Nhiều dịch vụ đám mây chọn không hiện thực giao dịch phân tán chính vì những vấn đề vận hành này[^82].

Một số hiện thực giao dịch phân tán gây tổn hao hiệu năng nặng nề. Chi phí cố hữu của commit hai pha phần lớn đến từ các lần ép ghi đĩa (`fsync`) thêm để khôi phục sau sập, và các vòng mạng thêm.

Tuy vậy, ta không nên gạt bỏ hoàn toàn giao dịch phân tán, mà nên xem xét kỹ và rút ra bài học quan trọng. Trước hết cần làm rõ "giao dịch phân tán" thực sự chỉ gì, vì người ta thường lẫn lộn hai loại giao dịch phân tán hoàn toàn khác nhau:

Giao dịch phân tán nội bộ cơ sở dữ liệu
: Một số cơ sở dữ liệu phân tán (tức cơ sở dữ liệu dùng sao chép (*replication*) và sharding ngay trong cấu hình chuẩn) hỗ trợ giao dịch nội bộ giữa các node cơ sở dữ liệu. Ví dụ, YugabyteDB, TiDB, FoundationDB, Spanner, VoltDB và storage engine NDB của MySQL Cluster đều hỗ trợ giao dịch nội bộ. Khi đó, mọi node tham gia giao dịch đều chạy cùng một phần mềm cơ sở dữ liệu.

Giao dịch phân tán không đồng nhất
: Giao dịch *không đồng nhất* (*heterogeneous*) có các participant thuộc hai hoặc nhiều công nghệ khác nhau, ví dụ hai cơ sở dữ liệu của hai hãng khác nhau, thậm chí cả hệ thống không phải cơ sở dữ liệu như message broker. Dù các hệ thống bên dưới hoàn toàn khác nhau, giao dịch phân tán xuyên hệ thống vẫn phải bảo đảm commit nguyên tử (*atomic commitment*).

Giao dịch nội bộ cơ sở dữ liệu không cần tương thích với hệ thống khác, nên có thể dùng giao thức tùy ý và tối ưu cho công nghệ cụ thể. Vì vậy giao dịch phân tán nội bộ cơ sở dữ liệu thường hoạt động tốt, còn giao dịch xuyên các công nghệ không đồng nhất khó hơn nhiều.

#### Xử lý tin nhắn đúng một lần {#sec_transactions_exactly_once}

Giao dịch phân tán không đồng nhất có thể tích hợp các hệ thống khác nhau theo cách mạnh mẽ. Ví dụ, chỉ xác nhận một tin nhắn trong hàng đợi tin nhắn là đã xử lý khi và chỉ khi giao dịch cơ sở dữ liệu xử lý tin nhắn đó commit thành công. Cách làm là commit nguyên tử (*atomic commitment*) *xác nhận tin nhắn* và *thao tác ghi cơ sở dữ liệu* trong cùng một giao dịch. Với giao dịch phân tán, điều này làm được ngay cả khi message broker và cơ sở dữ liệu là hai công nghệ không liên quan, chạy trên các máy khác nhau.

Nếu việc chuyển tin nhắn hoặc giao dịch cơ sở dữ liệu thất bại, cả hai đều abort, và message broker sau đó có thể giao lại an toàn. Bằng cách commit nguyên tử (*atomic commitment*) tin nhắn cùng tác dụng phụ của việc xử lý nó, ta bảo đảm tin nhắn thực tế được *xử lý đúng một lần*, dù trước khi thành công có thử lại nhiều lần. Mỗi lần abort đều loại bỏ mọi tác dụng phụ của giao dịch chưa hoàn tất. Đó là *ngữ nghĩa exactly-once* (*exactly-once semantics*).

Tuy nhiên, loại giao dịch phân tán này chỉ khả thi nếu mọi hệ thống mà giao dịch tác động đều dùng được cùng một giao thức commit nguyên tử (*atomic commitment*). Ví dụ, giả sử tác dụng phụ của việc xử lý tin nhắn là gửi email, mà máy chủ email không hỗ trợ commit hai pha, nếu xử lý tin nhắn thất bại và thử lại, email có thể bị gửi hai lần hoặc nhiều hơn. Ngược lại, nếu mọi tác dụng phụ của việc xử lý tin nhắn đều rollback được khi giao dịch abort, việc xử lý có thể thử lại an toàn như chưa từng có gì xảy ra.

Chương này sau đó sẽ quay lại ngữ nghĩa exactly-once. Giờ hãy xem các giao thức commit nguyên tử (*atomic commitment*) hỗ trợ loại giao dịch phân tán không đồng nhất này.

#### Giao dịch XA {#xa-transactions}

*X/Open XA* (viết tắt của *eXtended Architecture*, kiến trúc mở rộng) là chuẩn để hiện thực commit hai pha xuyên các công nghệ không đồng nhất[^73]. Nó ra đời năm 1991 và nay được hiện thực rộng rãi: nhiều cơ sở dữ liệu quan hệ truyền thống (gồm PostgreSQL, MySQL, Db2, SQL Server và Oracle) và message broker (gồm ActiveMQ, HornetQ, MSMQ và IBM MQ) đều hỗ trợ XA.

XA không phải giao thức mạng, mà chỉ là một bộ C API để tương tác với coordinator giao dịch. Các ngôn ngữ khác cũng có binding tương ứng. Ví dụ, trong ứng dụng Java EE, giao dịch XA được hiện thực qua Java Transaction API (JTA), được hỗ trợ bởi nhiều driver cơ sở dữ liệu dùng Java Database Connectivity (JDBC) và driver message broker dùng API Java Message Service (JMS).

XA giả định ứng dụng giao tiếp với cơ sở dữ liệu hoặc dịch vụ nhắn tin của participant qua driver mạng hoặc thư viện client. Driver hỗ trợ XA nghĩa là nó gọi XA API để xác định một thao tác có thuộc giao dịch phân tán không. Nếu có, nó gửi thông tin cần thiết cho máy chủ cơ sở dữ liệu. Driver cũng phơi ra các callback để coordinator yêu cầu participant prepare, commit hoặc abort.

Coordinator giao dịch hiện thực XA API. Chuẩn không quy định cách hiện thực cụ thể. Trong thực tế, coordinator thường chỉ là một thư viện được nạp vào cùng tiến trình với ứng dụng khởi tạo giao dịch, chứ không phải dịch vụ độc lập. Nó theo dõi mọi participant của giao dịch, yêu cầu họ prepare rồi thu phản hồi qua callback của driver, và dùng log trên đĩa cục bộ để ghi quyết định commit hay abort của từng giao dịch.

Nếu tiến trình ứng dụng sập, hoặc máy chạy ứng dụng chết, coordinator cũng biến mất theo. Mọi participant đang giữ giao dịch đã prepare nhưng chưa commit đều rơi vào trạng thái in doubt. Log của coordinator nằm trên đĩa cục bộ của máy chủ ứng dụng, nên phải khởi động lại máy chủ này, rồi thư viện coordinator đọc log để khôi phục kết quả commit hay abort của từng giao dịch. Sau đó coordinator mới yêu cầu được participant commit hoặc abort qua callback XA của driver cơ sở dữ liệu. Máy chủ cơ sở dữ liệu không thể liên lạc trực tiếp với coordinator, vì mọi giao tiếp đều phải đi qua thư viện client.

#### Giữ khóa khi in doubt {#holding-locks-while-in-doubt}

Sao phải để tâm đến giao dịch in doubt đến vậy? Chẳng lẽ phần còn lại của hệ thống không thể cứ chạy bình thường, tạm bỏ qua các giao dịch mà cuối cùng rồi cũng được dọn?

Vấn đề nằm ở *khóa*. Như đã nói ở ["Read committed"](/ch8#sec_transactions_read_committed), giao dịch cơ sở dữ liệu thường lấy khóa độc quyền cấp dòng trên mỗi dòng nó sửa để ngăn dirty write. Nếu còn yêu cầu cô lập serializable, cơ sở dữ liệu dùng khóa hai pha cũng phải lấy khóa chia sẻ trên mọi dòng mà giao dịch *đọc*.

Cơ sở dữ liệu không thể nhả các khóa này trước khi giao dịch commit hoặc abort (như vùng tô bóng trong {{< xref fig="8-13" page="/ch8" anchor="fig_transactions_two_phase_commit" >}}Hình 8-13{{< /xref >}}). Vì vậy, khi dùng commit hai pha, giao dịch phải giữ khóa suốt thời gian in doubt. Nếu coordinator sập rồi mất 20 phút để khởi động lại, khóa sẽ bị giữ 20 phút, còn nếu log của coordinator mất hẳn vì lý do nào đó, khóa thậm chí bị giữ vĩnh viễn — ít nhất cho đến khi quản trị viên giải quyết thủ công.

Trong lúc khóa bị giữ, giao dịch khác không sửa được các dòng này. Tùy mức cô lập, thậm chí việc đọc cũng có thể bị chặn. Do đó, các giao dịch khác không thể cứ thế chạy tiếp — hễ truy cập cùng dữ liệu là bị kẹt. Phần lớn chức năng của ứng dụng có thể không dùng được cho đến khi giao dịch in doubt được giải quyết.

#### Khôi phục sau khi coordinator hỏng {#recovering-from-coordinator-failure}

Về lý thuyết, khi coordinator sập rồi khởi động lại, nó phải khôi phục đầy đủ trạng thái từ log và giải quyết mọi giao dịch in doubt. Nhưng trong thực tế vẫn xuất hiện giao dịch in doubt *mồ côi* (*orphaned*)[^83] [^84] — coordinator vì lý do nào đó không xác định được kết quả giao dịch, ví dụ do bug phần mềm làm log giao dịch bị mất hoặc hỏng. Các giao dịch này không giải quyết tự động được, đành nằm mãi trong cơ sở dữ liệu, giữ khóa và chặn các giao dịch khác.

Khởi động lại máy chủ cơ sở dữ liệu cũng không giải quyết được. Hiện thực 2PC đúng đắn phải giữ khóa của giao dịch in doubt qua các lần khởi động lại, nếu không có nguy cơ phá vỡ bảo đảm tính nguyên tử. Tình huống này rất hóc búa.

Lối thoát duy nhất là quản trị viên thủ công quyết định commit hay rollback. Quản trị viên phải kiểm tra mọi participant của từng giao dịch in doubt, xác định đã có participant nào commit hoặc abort chưa, rồi áp dụng cùng kết quả cho các participant còn lại. Việc này có thể tốn rất nhiều sức người, và thường phải làm trong lúc sự cố sản xuất nghiêm trọng, dưới áp lực tinh thần và thời gian rất lớn — nếu không thì coordinator đã không rơi vào tình trạng tồi tệ như vậy.

Nhiều hiện thực XA có một lối thoát khẩn cấp gọi là *quyết định heuristic* (*heuristic decision*): cho phép participant đơn phương abort hoặc commit giao dịch in doubt dù coordinator chưa đưa ra quyết định rõ ràng[^73]. Nói thẳng, "*heuristic*" ở đây chỉ là cách nói giảm của "*rất có thể phá vỡ tính nguyên tử*", vì nó vi phạm hệ thống lời hứa của commit hai pha. Vì vậy, quyết định heuristic chỉ dùng để thoát khỏi tình huống thảm họa, tuyệt đối không phải biện pháp thường dùng.

#### Các vấn đề của giao dịch XA {#problems-with-xa-transactions}

Coordinator đơn node là điểm lỗi đơn của toàn hệ thống. Đặt coordinator trong máy chủ ứng dụng cũng có vấn đề, vì log trên đĩa cục bộ của coordinator trở thành phần then chốt của trạng thái bền vững của hệ thống, quan trọng không kém chính cơ sở dữ liệu.

Về nguyên tắc, coordinator của giao dịch XA có thể được làm sẵn sàng cao và sao chép (*replication*) như các cơ sở dữ liệu quan trọng khác. Đáng tiếc, điều này vẫn không giải quyết được một vấn đề căn bản của XA: nó không cung cấp cách để coordinator và participant của giao dịch liên lạc trực tiếp. Chúng chỉ liên lạc qua code ứng dụng khởi tạo giao dịch và driver cơ sở dữ liệu mà ứng dụng dùng để gọi participant.

Do đó, dù coordinator được sao chép (*replication*), code ứng dụng vẫn là điểm lỗi đơn. Muốn giải quyết, phải thiết kế lại hoàn toàn cách code ứng dụng chạy, để nó sao chép (*replication*) hoặc khởi động lại được, về hình thức có thể giống durable execution (xem ["Durable execution và workflow"](/ch5#sec_encoding_dataflow_workflows)). Nhưng thực tế dường như chưa có công cụ nào thật sự áp dụng cách này.

Một vấn đề khác là XA phải tương thích với đủ loại hệ thống dữ liệu nên chỉ có thể lấy "mẫu số chung nhỏ nhất". Ví dụ, nó không phát hiện được deadlock xuyên các hệ thống khác nhau, vì việc đó cần một giao thức chuẩn để các hệ thống trao đổi xem mỗi giao dịch đang chờ khóa nào, và nó cũng không dùng chung được với SSI (xem ["Serializable snapshot isolation (SSI)"](/ch8#sec_transactions_ssi)), vì SSI cần giao thức nhận diện xung đột xuyên hệ thống.

Một phần các vấn đề này là khó khăn cố hữu của việc thực hiện giao dịch xuyên các công nghệ không đồng nhất. Tuy nhiên, giữ cho nhiều hệ thống dữ liệu không đồng nhất nhất quán với nhau vẫn là nhu cầu thật và quan trọng, nên phải tìm giải pháp khác. Giải pháp có tồn tại, sẽ được giới thiệu ở mục tiếp theo và ở ["Dữ liệu dẫn xuất và giao dịch phân tán"](/ch13#sec_future_derived_vs_transactions).

### Giao dịch phân tán nội bộ cơ sở dữ liệu {#sec_transactions_internal}

Như đã nói, giao dịch phân tán xuyên nhiều công nghệ lưu trữ không đồng nhất rất khác với giao dịch phân tán trong nội bộ một hệ thống, nơi mọi node tham gia là các shard của cùng một cơ sở dữ liệu, chạy cùng phần mềm. Giao dịch phân tán nội bộ là đặc điểm nổi bật của các cơ sở dữ liệu "NewSQL" như CockroachDB[^5], TiDB[^6], Spanner[^7], FoundationDB[^8] và YugabyteDB. Các message broker như Kafka cũng hỗ trợ giao dịch phân tán nội bộ[^85].

Nhiều hệ thống trong số này dùng commit hai pha để bảo đảm tính nguyên tử của giao dịch ghi nhiều shard, mà không gặp các vấn đề của giao dịch XA. Vì giao dịch phân tán của chúng không cần giao tiếp với công nghệ khác, chúng tránh được cái bẫy mẫu số chung nhỏ nhất, người thiết kế hệ thống tự do dùng giao thức đáng tin cậy và nhanh hơn.

Những vấn đề nghiêm trọng nhất của XA có thể giải quyết như sau:

* Sao chép (*replication*) coordinator: khi coordinator chính sập thì tự động failover sang node coordinator khác;
* Cho phép coordinator và các shard dữ liệu giao tiếp trực tiếp, không đi qua code ứng dụng;
* Sao chép (*replication*) các shard tham gia giao dịch, giảm nguy cơ phải abort giao dịch vì một shard lỗi;
* Kết hợp giao thức commit nguyên tử (*atomic commitment*) với giao thức kiểm soát đồng thời phân tán, để giao thức sau hỗ trợ phát hiện deadlock xuyên shard và đọc nhất quán.

Coordinator và các shard cơ sở dữ liệu thường được sao chép (*replication*) bằng thuật toán đồng thuận. [Chương 10](/ch10#ch_consistency) sẽ giới thiệu cách dùng thuật toán đồng thuận để hiện thực commit nguyên tử (*atomic commitment*) cho giao dịch phân tán. Các thuật toán này tự động chuyển từ node lỗi sang node khác mà không cần can thiệp thủ công, vừa chịu được lỗi vừa tiếp tục bảo đảm các thuộc tính nhất quán mạnh.

Mức cô lập mà giao dịch phân tán cung cấp tùy từng hệ thống cụ thể, nhưng hiện thực snapshot isolation và serializable snapshot isolation xuyên shard đều khả thi. Cách hoạt động được trình bày chi tiết trong các bài báo trích dẫn ở cuối chương.

#### Xử lý tin nhắn đúng một lần, nhìn lại {#exactly-once-message-processing-revisited}

["Xử lý tin nhắn đúng một lần"](/ch8#sec_transactions_exactly_once) đã giới thiệu một công dụng quan trọng của giao dịch phân tán: bảo đảm một thao tác có hiệu lực đúng một lần, dù trong quá trình xử lý có sập và phải thử lại. Nếu commit nguyên tử (*atomic commitment*) được giao dịch xuyên message broker và cơ sở dữ liệu, ta có thể làm được: chỉ xác nhận tin nhắn với broker khi và chỉ khi tin nhắn được xử lý thành công và thao tác ghi cơ sở dữ liệu sinh ra từ việc xử lý cũng đã commit thành công.

Tuy nhiên, để đạt ngữ nghĩa exactly-once thực ra không cần giao dịch phân tán như vậy. Phương án thay thế dưới đây chỉ đòi hỏi bản thân cơ sở dữ liệu hỗ trợ giao dịch:

1. Giả sử mỗi tin nhắn có ID duy nhất, và cơ sở dữ liệu có một bảng ghi lại ID các tin nhắn đã xử lý. Khi nhận tin nhắn từ broker và bắt đầu xử lý, trước hết bắt đầu một giao dịch mới trong cơ sở dữ liệu rồi kiểm tra ID tin nhắn. Nếu cơ sở dữ liệu đã có cùng ID, ta biết tin nhắn đã xử lý, có thể xác nhận với broker và bỏ tin nhắn này.
2. Nếu cơ sở dữ liệu chưa có ID tin nhắn, thêm nó vào bảng rồi xử lý tin nhắn. Quá trình xử lý có thể ghi thêm vào cơ sở dữ liệu trong cùng giao dịch. Khi xử lý xong, commit giao dịch cơ sở dữ liệu.
3. Sau khi giao dịch cơ sở dữ liệu commit thành công, có thể xác nhận tin nhắn với broker.
4. Sau khi xác nhận tin nhắn với broker thành công, ta biết broker sẽ không thử xử lý lại cùng tin nhắn đó, nên có thể mở giao dịch khác để xóa ID tin nhắn khỏi cơ sở dữ liệu.

Nếu bộ xử lý tin nhắn sập trước khi commit giao dịch cơ sở dữ liệu, giao dịch bị abort và message broker sau đó thử lại. Nếu nó sập sau khi commit nhưng trước khi xác nhận với broker, broker cũng thử lại, nhưng khi thử lại sẽ thấy ID tin nhắn trong cơ sở dữ liệu nên bỏ tin nhắn đó. Nếu nó sập sau khi xác nhận nhưng trước khi xóa ID tin nhắn khỏi cơ sở dữ liệu, cơ sở dữ liệu chỉ còn sót một ID tin nhắn cũ, ngoài việc chiếm chút không gian thì không gây hại gì. Việc thử lại cũng có thể xảy ra trước khi giao dịch cơ sở dữ liệu ban đầu abort — ví dụ khi liên lạc giữa bộ xử lý tin nhắn và cơ sở dữ liệu bị gián đoạn — khi đó ràng buộc duy nhất trên bảng ID tin nhắn sẽ ngăn hai giao dịch đồng thời chèn cùng một ID.

Như vậy, hiện thực xử lý đúng một lần chỉ cần giao dịch trong nội bộ cơ sở dữ liệu. Trường hợp này không đòi hỏi tính nguyên tử giữa cơ sở dữ liệu và message broker. Việc ghi ID tin nhắn vào cơ sở dữ liệu làm cho việc xử lý tin nhắn có tính *idempotence* (*idempotence*), nhờ đó thử lại an toàn mà không sinh lặp tác dụng phụ. Các framework xử lý luồng như Kafka Streams cũng dùng cách tương tự để hiện thực ngữ nghĩa exactly-once, xem chi tiết ở ["Fault Tolerance"](/ch12#sec_stream_fault_tolerance).

Tuy vậy, giao dịch phân tán nội bộ cơ sở dữ liệu vẫn giúp mở rộng mẫu này. Ví dụ, ID tin nhắn có thể lưu trên một shard, còn dữ liệu nghiệp vụ mà việc xử lý tin nhắn cập nhật nằm ở các shard khác, rồi để giao dịch nội bộ bảo đảm commit xuyên shard có tính nguyên tử.



## Tóm tắt {#summary}

Giao dịch là một lớp trừu tượng cho phép ứng dụng giả vờ rằng một số vấn đề đồng thời và một số lỗi phần mềm, phần cứng không tồn tại. Rất nhiều loại lỗi được đơn giản hóa thành một lần *giao dịch abort*, và ứng dụng chỉ cần thử lại.

Chương này đã xem xét nhiều vấn đề mà giao dịch giúp phòng tránh. Không phải ứng dụng nào cũng gặp đủ mọi vấn đề đó: ứng dụng có mẫu truy cập rất đơn giản (như chỉ đọc ghi từng bản ghi) có lẽ vẫn xoay xở được mà không cần giao dịch. Nhưng với các mẫu truy cập phức tạp hơn, giao dịch có thể giảm đáng kể số kịch bản lỗi tiềm ẩn cần cân nhắc.

Nếu không có giao dịch, các kịch bản lỗi như tiến trình sập, mạng đứt, mất điện, đầy đĩa, đồng thời bất ngờ… đều có thể khiến dữ liệu không nhất quán theo nhiều cách. Chẳng hạn, dữ liệu phi chuẩn hóa rất dễ lệch khỏi dữ liệu nguồn. Thiếu giao dịch, rất khó suy luận hậu quả mà các truy cập phức tạp, ảnh hưởng lẫn nhau, gây ra cho cơ sở dữ liệu.

Chương này đi sâu vào kiểm soát đồng thời. Ta đã giới thiệu một số mức cô lập được dùng rộng rãi, đặc biệt là *read committed*, *snapshot isolation* (đôi khi gọi là *repeatable read*) và *serializable*, và mô tả chúng qua các race condition khác nhau. {{< xref tbl="8-1" page="/ch8" anchor="ch_transactions_isolation_levels" >}}Bảng 8-1{{< /xref >}} tóm tắt nội dung này:

| Mức cô lập | Dirty read | Read skew | Phantom | Lost update | Write skew |
|------|------|------|------|-------|------|
| Read uncommitted | ✗ Có thể | ✗ Có thể | ✗ Có thể | ✗ Có thể | ✗ Có thể |
| Read committed | ✓ Ngăn được | ✗ Có thể | ✗ Có thể | ✗ Có thể | ✗ Có thể |
| Snapshot isolation | ✓ Ngăn được | ✓ Ngăn được | ✓ Ngăn được | ? Tùy trường hợp | ✗ Có thể |
| Serializable | ✓ Ngăn được | ✓ Ngăn được | ✓ Ngăn được | ✓ Ngăn được | ✓ Ngăn được |
{#ch_transactions_isolation_levels num="8-1" caption="Tóm tắt các bất thường có thể xảy ra ở từng mức cô lập." .wide .expand}

Dirty read
: Một client đọc được dữ liệu do client khác ghi trước khi nó được commit. Read committed và các mức cô lập mạnh hơn ngăn được dirty read.

Dirty write
: Một client ghi đè dữ liệu mà client khác đã ghi nhưng chưa commit. Hầu như mọi hiện thực giao dịch đều ngăn dirty write.

Read skew
: Client quan sát các phần khác nhau của cơ sở dữ liệu ở những thời điểm khác nhau. Một số dạng read skew còn gọi là *nonrepeatable read*. Cách phổ biến nhất để ngăn là snapshot isolation, cho phép giao dịch đọc một snapshot nhất quán ứng với một thời điểm cụ thể, thường hiện thực bằng *kiểm soát đồng thời đa phiên bản* (MVCC).

Lost update
: Hai client đồng thời chạy chu trình read-modify-write, một bên ghi đè thao tác ghi của bên kia mà không gộp thay đổi của bên kia, gây mất dữ liệu. Một số hiện thực snapshot isolation tự động ngăn bất thường này, số khác cần khóa thủ công (`SELECT FOR UPDATE`).

Write skew
: Giao dịch đọc dữ liệu, ra quyết định dựa trên giá trị đã thấy, rồi ghi quyết định đó vào cơ sở dữ liệu. Nhưng đến lúc ghi, tiền đề của quyết định không còn đúng. Chỉ cô lập serializable ngăn được bất thường này.

Phantom
: Giao dịch đọc các đối tượng thỏa một điều kiện tìm kiếm, rồi client khác ghi làm thay đổi kết quả tìm kiếm. Snapshot isolation ngăn được phantom trực tiếp, nhưng phantom trong tình huống write skew cần xử lý đặc biệt, ví dụ dùng khóa phạm vi chỉ mục (*index*).

Các mức cô lập yếu ngăn được một số bất thường, nhưng để lại các bất thường khác cho lập trình viên ứng dụng xử lý thủ công (ví dụ khóa tường minh). Chỉ cô lập serializable ngăn được mọi vấn đề này. Ta đã bàn ba cách hiện thực giao dịch serializable:

Thực thi giao dịch đúng theo thứ tự tuần tự
: Nếu mỗi giao dịch đều hoàn tất cực nhanh (thường nhờ thủ tục lưu trữ), và thông lượng giao dịch đủ thấp để một lõi CPU xử lý được, hoặc giao dịch có thể chia shard, thì đây là lựa chọn đơn giản và hiệu quả.

Khóa hai pha
: Suốt nhiều thập kỷ đây là cách chuẩn để hiện thực serializable, nhưng do hiệu năng kém nên nhiều ứng dụng tránh dùng.

Serializable snapshot isolation (SSI)
: Thuật toán tương đối mới, tránh được phần lớn nhược điểm của hai phương án trước. Nó theo hướng lạc quan, cho giao dịch tiếp tục chạy mà không bị chặn, khi giao dịch sắp commit mới bị kiểm tra, và nếu kết quả thực thi không serializable thì bị abort.

Cuối cùng, ta đã xem xét cách dùng commit hai pha để hiện thực tính nguyên tử cho giao dịch xuyên nhiều node. Nếu mọi node chạy cùng một phần mềm cơ sở dữ liệu, giao dịch phân tán thường hoạt động tốt, nhưng khi xuyên các công nghệ lưu trữ khác nhau (dùng giao dịch XA), 2PC gặp nhiều vấn đề: nó rất nhạy với lỗi ở coordinator và ở code ứng dụng điều khiển giao dịch, và khó phối hợp với các cơ chế kiểm soát đồng thời. May thay, idempotence có thể bảo đảm ngữ nghĩa exactly-once mà không đòi hỏi commit nguyên tử (*atomic commitment*) xuyên các kho lưu trữ không đồng nhất. Các chương sau sẽ tiếp tục bàn chủ đề này.

Ví dụ trong chương này dùng mô hình dữ liệu quan hệ. Tuy nhiên, như đã nói ở ["Nhu cầu về giao dịch đa đối tượng"](/ch8#sec_transactions_need), giao dịch là tính năng cơ sở dữ liệu đáng giá bất kể dùng mô hình dữ liệu nào.



### Tài liệu tham khảo {#references}

[^1]: Steven J. Murdoch. [What went wrong with Horizon: learning from the Post Office Trial](https://www.benthamsgaze.org/2021/07/15/what-went-wrong-with-horizon-learning-from-the-post-office-trial/). *benthamsgaze.org*, July 2021. Archived at [perma.cc/CNM4-553F](https://perma.cc/CNM4-553F)
[^2]: Donald D. Chamberlin, Morton M. Astrahan, Michael W. Blasgen, James N. Gray, W. Frank King, Bruce G. Lindsay, Raymond Lorie, James W. Mehl, Thomas G. Price, Franco Putzolu, Patricia Griffiths Selinger, Mario Schkolnick, Donald R. Slutz, Irving L. Traiger, Bradford W. Wade, and Robert A. Yost. [A History and Evaluation of System R](https://dsf.berkeley.edu/cs262/2005/SystemR.pdf). *Communications of the ACM*, volume 24, issue 10, pages 632–646, October 1981. [doi:10.1145/358769.358784](https://doi.org/10.1145/358769.358784)
[^3]: Jim N. Gray, Raymond A. Lorie, Gianfranco R. Putzolu, and Irving L. Traiger. [Granularity of Locks and Degrees of Consistency in a Shared Data Base](https://citeseerx.ist.psu.edu/pdf/e127f0a6a912bb9150ecfe03c0ebf7fbc289a023). in *Modelling in Data Base Management Systems: Proceedings of the IFIP Working Conference on Modelling in Data Base Management Systems*, edited by G. M. Nijssen, pages 364–394, Elsevier/North Holland Publishing, 1976. Also in *Readings in Database Systems*, 4th edition, edited by Joseph M. Hellerstein and Michael Stonebraker, MIT Press, 2005. ISBN: 978-0-262-69314-1
[^4]: Kapali P. Eswaran, Jim N. Gray, Raymond A. Lorie, and Irving L. Traiger. [The Notions of Consistency and Predicate Locks in a Database System](https://jimgray.azurewebsites.net/papers/On%20the%20Notions%20of%20Consistency%20and%20Predicate%20Locks%20in%20a%20Database%20System%20CACM.pdf?from=https://research.microsoft.com/en-us/um/people/gray/papers/On%20the%20Notions%20of%20Consistency%20and%20Predicate%20Locks%20in%20a%20Database%20System%20CACM.pdf). *Communications of the ACM*, volume 19, issue 11, pages 624–633, November 1976. [doi:10.1145/360363.360369](https://doi.org/10.1145/360363.360369)
[^5]: Rebecca Taft, Irfan Sharif, Andrei Matei, Nathan VanBenschoten, Jordan Lewis, Tobias Grieger, Kai Niemi, Andy Woods, Anne Birzin, Raphael Poss, Paul Bardea, Amruta Ranade, Ben Darnell, Bram Gruneir, Justin Jaffray, Lucy Zhang, and Peter Mattis. [CockroachDB: The Resilient Geo-Distributed SQL Database](https://dl.acm.org/doi/pdf/10.1145/3318464.3386134). At *ACM SIGMOD International Conference on Management of Data* (SIGMOD), pages 1493–1509, June 2020. [doi:10.1145/3318464.3386134](https://doi.org/10.1145/3318464.3386134)
[^6]: Dongxu Huang, Qi Liu, Qiu Cui, Zhuhe Fang, Xiaoyu Ma, Fei Xu, Li Shen, Liu Tang, Yuxing Zhou, Menglong Huang, Wan Wei, Cong Liu, Jian Zhang, Jianjun Li, Xuelian Wu, Lingyu Song, Ruoxi Sun, Shuaipeng Yu, Lei Zhao, Nicholas Cameron, Liquan Pei, and Xin Tang. [TiDB: a Raft-based HTAP database](https://www.vldb.org/pvldb/vol13/p3072-huang.pdf). *Proceedings of the VLDB Endowment*, volume 13, issue 12, pages 3072–3084. [doi:10.14778/3415478.3415535](https://doi.org/10.14778/3415478.3415535)
[^7]: James C. Corbett, Jeffrey Dean, Michael Epstein, Andrew Fikes, Christopher Frost, JJ Furman, Sanjay Ghemawat, Andrey Gubarev, Christopher Heiser, Peter Hochschild, Wilson Hsieh, Sebastian Kanthak, Eugene Kogan, Hongyi Li, Alexander Lloyd, Sergey Melnik, David Mwaura, David Nagle, Sean Quinlan, Rajesh Rao, Lindsay Rolig, Dale Woodford, Yasushi Saito, Christopher Taylor, Michal Szymaniak, and Ruth Wang. [Spanner: Google’s Globally-Distributed Database](https://research.google/pubs/pub39966/). At *10th USENIX Symposium on Operating System Design and Implementation* (OSDI), October 2012.
[^8]: Jingyu Zhou, Meng Xu, Alexander Shraer, Bala Namasivayam, Alex Miller, Evan Tschannen, Steve Atherton, Andrew J. Beamon, Rusty Sears, John Leach, Dave Rosenthal, Xin Dong, Will Wilson, Ben Collins, David Scherer, Alec Grieser, Young Liu, Alvin Moore, Bhaskar Muppana, Xiaoge Su, and Vishesh Yadav. [FoundationDB: A Distributed Unbundled Transactional Key Value Store](https://www.foundationdb.org/files/fdb-paper.pdf). At *ACM International Conference on Management of Data* (SIGMOD), June 2021. [doi:10.1145/3448016.3457559](https://doi.org/10.1145/3448016.3457559)
[^9]: Theo Härder and Andreas Reuter. [Principles of Transaction-Oriented Database Recovery](https://citeseerx.ist.psu.edu/pdf/11ef7c142295aeb1a28a0e714c91fc8d610c3047). *ACM Computing Surveys*, volume 15, issue 4, pages 287–317, December 1983. [doi:10.1145/289.291](https://doi.org/10.1145/289.291)
[^10]: Peter Bailis, Alan Fekete, Ali Ghodsi, Joseph M. Hellerstein, and Ion Stoica. [HAT, not CAP: Towards Highly Available Transactions](https://www.usenix.org/system/files/conference/hotos13/hotos13-final80.pdf). At *14th USENIX Workshop on Hot Topics in Operating Systems* (HotOS), May 2013.
[^11]: Armando Fox, Steven D. Gribble, Yatin Chawathe, Eric A. Brewer, and Paul Gauthier. [Cluster-Based Scalable Network Services](https://people.eecs.berkeley.edu/~brewer/cs262b/TACC.pdf). At *16th ACM Symposium on Operating Systems Principles* (SOSP), October 1997. [doi:10.1145/268998.266662](https://doi.org/10.1145/268998.266662)
[^12]: Tony Andrews. [Enforcing Complex Constraints in Oracle](https://tonyandrews.blogspot.com/2004/10/enforcing-complex-constraints-in.html). *tonyandrews.blogspot.co.uk*, October 2004. Archived at [archive.org](https://web.archive.org/web/20220201190625/https%3A//tonyandrews.blogspot.com/2004/10/enforcing-complex-constraints-in.html)
[^13]: Philip A. Bernstein, Vassos Hadzilacos, and Nathan Goodman. [*Concurrency Control and Recovery in Database Systems*](https://www.microsoft.com/en-us/research/people/philbe/book/). Addison-Wesley, 1987. ISBN: 978-0-201-10715-9, available online at [*microsoft.com*](https://www.microsoft.com/en-us/research/people/philbe/book/).
[^14]: Alan Fekete, Dimitrios Liarokapis, Elizabeth O’Neil, Patrick O’Neil, and Dennis Shasha. [Making Snapshot Isolation Serializable](https://www.cse.iitb.ac.in/infolab/Data/Courses/CS632/2009/Papers/p492-fekete.pdf). *ACM Transactions on Database Systems*, volume 30, issue 2, pages 492–528, June 2005. [doi:10.1145/1071610.1071615](https://doi.org/10.1145/1071610.1071615)
[^15]: Mai Zheng, Joseph Tucek, Feng Qin, and Mark Lillibridge. [Understanding the Robustness of SSDs Under Power Fault](https://www.usenix.org/system/files/conference/fast13/fast13-final80.pdf). At *11th USENIX Conference on File and Storage Technologies* (FAST), February 2013.
[^16]: Laurie Denness. [SSDs: A Gift and a Curse](https://laur.ie/blog/2015/06/ssds-a-gift-and-a-curse/). *laur.ie*, June 2015. Archived at [perma.cc/6GLP-BX3T](https://perma.cc/6GLP-BX3T)
[^17]: Adam Surak. [When Solid State Drives Are Not That Solid](https://www.algolia.com/blog/engineering/when-solid-state-drives-are-not-that-solid). *blog.algolia.com*, June 2015. Archived at [perma.cc/CBR9-QZEE](https://perma.cc/CBR9-QZEE)
[^18]: Hewlett Packard Enterprise. [Bulletin: (Revision) HPE SAS Solid State Drives - Critical Firmware Upgrade Required for Certain HPE SAS Solid State Drive Models to Prevent Drive Failure at 32,768 Hours of Operation](https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-a00092491en_us). *support.hpe.com*, November 2019. Archived at [perma.cc/CZR4-AQBS](https://perma.cc/CZR4-AQBS)
[^19]: Craig Ringer et al. [PostgreSQL’s handling of fsync() errors is unsafe and risks data loss at least on XFS](https://www.postgresql.org/message-id/flat/CAMsr%2BYHh%2B5Oq4xziwwoEfhoTZgr07vdGG%2Bhu%3D1adXx59aTeaoQ%40mail.gmail.com). Email thread on pgsql-hackers mailing list, *postgresql.org*, March 2018. Archived at [perma.cc/5RKU-57FL](https://perma.cc/5RKU-57FL)
[^20]: Anthony Rebello, Yuvraj Patel, Ramnatthan Alagappan, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. [Can Applications Recover from fsync Failures?](https://www.usenix.org/conference/atc20/presentation/rebello) At *USENIX Annual Technical Conference* (ATC), July 2020.
[^21]: Thanumalayan Sankaranarayana Pillai, Vijay Chidambaram, Ramnatthan Alagappan, Samer Al-Kiswany, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. [Crash Consistency: Rethinking the Fundamental Abstractions of the File System](https://dl.acm.org/doi/pdf/10.1145/2800695.2801719). *ACM Queue*, volume 13, issue 7, pages 20–28, July 2015. [doi:10.1145/2800695.2801719](https://doi.org/10.1145/2800695.2801719)
[^22]: Thanumalayan Sankaranarayana Pillai, Vijay Chidambaram, Ramnatthan Alagappan, Samer Al-Kiswany, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. [All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-pillai.pdf). At *11th USENIX Symposium on Operating Systems Design and Implementation* (OSDI), October 2014.
[^23]: Chris Siebenmann. [Unix’s File Durability Problem](https://utcc.utoronto.ca/~cks/space/blog/unix/FileSyncProblem). *utcc.utoronto.ca*, April 2016. Archived at [perma.cc/VSS8-5MC4](https://perma.cc/VSS8-5MC4)
[^24]: Aishwarya Ganesan, Ramnatthan Alagappan, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. [Redundancy Does Not Imply Fault Tolerance: Analysis of Distributed Storage Reactions to Single Errors and Corruptions](https://www.usenix.org/conference/fast17/technical-sessions/presentation/ganesan). At *15th USENIX Conference on File and Storage Technologies* (FAST), February 2017.
[^25]: Lakshmi N. Bairavasundaram, Garth R. Goodson, Bianca Schroeder, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. [An Analysis of Data Corruption in the Storage Stack](https://www.usenix.org/legacy/event/fast08/tech/full_papers/bairavasundaram/bairavasundaram.pdf). At *6th USENIX Conference on File and Storage Technologies* (FAST), February 2008.
[^26]: Bianca Schroeder, Raghav Lagisetty, and Arif Merchant. [Flash Reliability in Production: The Expected and the Unexpected](https://www.usenix.org/conference/fast16/technical-sessions/presentation/schroeder). At *14th USENIX Conference on File and Storage Technologies* (FAST), February 2016.
[^27]: Don Allison. [SSD Storage – Ignorance of Technology Is No Excuse](https://blog.korelogic.com/blog/2015/03/24). *blog.korelogic.com*, March 2015. Archived at [perma.cc/9QN4-9SNJ](https://perma.cc/9QN4-9SNJ)
[^28]: Gordon Mah Ung. [Debunked: Your SSD won’t lose data if left unplugged after all](https://www.pcworld.com/article/427602/debunked-your-ssd-wont-lose-data-if-left-unplugged-after-all.html). *pcworld.com*, May 2015. Archived at [perma.cc/S46H-JUDU](https://perma.cc/S46H-JUDU)
[^29]: Martin Kleppmann. [Hermitage: Testing the ‘I’ in ACID](https://martin.kleppmann.com/2014/11/25/hermitage-testing-the-i-in-acid.html). *martin.kleppmann.com*, November 2014. Archived at [perma.cc/KP2Y-AQGK](https://perma.cc/KP2Y-AQGK)
[^30]: Todd Warszawski and Peter Bailis. [ACIDRain: Concurrency-Related Attacks on Database-Backed Web Applications](http://www.bailis.org/papers/acidrain-sigmod2017.pdf). At *ACM International Conference on Management of Data* (SIGMOD), May 2017. [doi:10.1145/3035918.3064037](https://doi.org/10.1145/3035918.3064037)
[^31]: Tristan D’Agosta. [BTC Stolen from Poloniex](https://bitcointalk.org/index.php?topic=499580). *bitcointalk.org*, March 2014. Archived at [perma.cc/YHA6-4C5D](https://perma.cc/YHA6-4C5D)
[^32]: bitcointhief2. [How I Stole Roughly 100 BTC from an Exchange and How I Could Have Stolen More!](https://www.reddit.com/r/Bitcoin/comments/1wtbiu/how_i_stole_roughly_100_btc_from_an_exchange_and/) *reddit.com*, February 2014. Archived at [archive.org](https://web.archive.org/web/20250118042610/https%3A//www.reddit.com/r/Bitcoin/comments/1wtbiu/how_i_stole_roughly_100_btc_from_an_exchange_and/)
[^33]: Sudhir Jorwekar, Alan Fekete, Krithi Ramamritham, and S. Sudarshan. [Automating the Detection of Snapshot Isolation Anomalies](https://www.vldb.org/conf/2007/papers/industrial/p1263-jorwekar.pdf). At *33rd International Conference on Very Large Data Bases* (VLDB), September 2007.
[^34]: Michael Melanson. [Transactions: The Limits of Isolation](https://www.michaelmelanson.net/posts/transactions-the-limits-of-isolation/). *michaelmelanson.net*, November 2014. Archived at [perma.cc/RG5R-KMYZ](https://perma.cc/RG5R-KMYZ)
[^35]: Edward Kim. [How ACH works: A developer perspective — Part 1](https://engineering.gusto.com/how-ach-works-a-developer-perspective-part-1-339d3e7bea1). *engineering.gusto.com*, April 2014. Archived at [perma.cc/7B2H-PU94](https://perma.cc/7B2H-PU94)
[^36]: Hal Berenson, Philip A. Bernstein, Jim N. Gray, Jim Melton, Elizabeth O’Neil, and Patrick O’Neil. [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf). At *ACM International Conference on Management of Data* (SIGMOD), May 1995. [doi:10.1145/568271.223785](https://doi.org/10.1145/568271.223785)
[^37]: Atul Adya. [Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed Transactions](https://pmg.csail.mit.edu/papers/adya-phd.pdf). PhD Thesis, Massachusetts Institute of Technology, March 1999. Archived at [perma.cc/E97M-HW5Q](https://perma.cc/E97M-HW5Q)
[^38]: Peter Bailis, Aaron Davidson, Alan Fekete, Ali Ghodsi, Joseph M. Hellerstein, and Ion Stoica. [Highly Available Transactions: Virtues and Limitations](https://www.vldb.org/pvldb/vol7/p181-bailis.pdf). At *40th International Conference on Very Large Data Bases* (VLDB), September 2014.
[^39]: Natacha Crooks, Youer Pu, Lorenzo Alvisi, and Allen Clement. [Seeing is Believing: A Client-Centric Specification of Database Isolation](https://www.cs.cornell.edu/lorenzo/papers/Crooks17Seeing.pdf). At *ACM Symposium on Principles of Distributed Computing* (PODC), pages 73–82, July 2017. [doi:10.1145/3087801.3087802](https://doi.org/10.1145/3087801.3087802)
[^40]: Bruce Momjian. [MVCC Unmasked](https://momjian.us/main/writings/pgsql/mvcc.pdf). *momjian.us*, July 2014. Archived at [perma.cc/KQ47-9GYB](https://perma.cc/KQ47-9GYB)
[^41]: Peter Alvaro and Kyle Kingsbury. [MySQL 8.0.34](https://jepsen.io/analyses/mysql-8.0.34). *jepsen.io*, December 2023. Archived at [perma.cc/HGE2-Z878](https://perma.cc/HGE2-Z878)
[^42]: Egor Rogov. [PostgreSQL 14 Internals](https://postgrespro.com/community/books/internals). *postgrespro.com*, April 2023. Archived at [perma.cc/FRK2-D7WB](https://perma.cc/FRK2-D7WB)
[^43]: Hironobu Suzuki. [The Internals of PostgreSQL](https://www.interdb.jp/pg/). *interdb.jp*, 2017.
[^44]: Rohan Reddy Alleti. [Internals of MVCC in Postgres: Hidden costs of Updates vs Inserts](https://medium.com/%40rohanjnr44/internals-of-mvcc-in-postgres-hidden-costs-of-updates-vs-inserts-381eadd35844). *medium.com*, March 2025. Archived at [perma.cc/3ACX-DFXT](https://perma.cc/3ACX-DFXT)
[^45]: Andy Pavlo and Bohan Zhang. [The Part of PostgreSQL We Hate the Most](https://www.cs.cmu.edu/~pavlo/blog/2023/04/the-part-of-postgresql-we-hate-the-most.html). *cs.cmu.edu*, April 2023. Archived at [perma.cc/XSP6-3JBN](https://perma.cc/XSP6-3JBN)
[^46]: Yingjun Wu, Joy Arulraj, Jiexi Lin, Ran Xian, and Andrew Pavlo. [An empirical evaluation of in-memory multi-version concurrency control](https://vldb.org/pvldb/vol10/p781-Wu.pdf). *Proceedings of the VLDB Endowment*, volume 10, issue 7, pages 781–792, March 2017. [doi:10.14778/3067421.3067427](https://doi.org/10.14778/3067421.3067427)
[^47]: Nikita Prokopov. [Unofficial Guide to Datomic Internals](https://tonsky.me/blog/unofficial-guide-to-datomic-internals/). *tonsky.me*, May 2014.
[^48]: Daniil Svetlov. [A Practical Guide to Taming Postgres Isolation Anomalies](https://dansvetlov.me/postgres-anomalies/). *dansvetlov.me*, March 2025. Archived at [perma.cc/L7LE-TDLS](https://perma.cc/L7LE-TDLS)
[^49]: Nate Wiger. [An Atomic Rant](https://nateware.com/2010/02/18/an-atomic-rant/). *nateware.com*, February 2010. Archived at [perma.cc/5ZYB-PE44](https://perma.cc/5ZYB-PE44)
[^50]: James Coglan. [Reading and writing, part 3: web applications](https://blog.jcoglan.com/2020/10/12/reading-and-writing-part-3/). *blog.jcoglan.com*, October 2020. Archived at [perma.cc/A7EK-PJVS](https://perma.cc/A7EK-PJVS)
[^51]: Peter Bailis, Alan Fekete, Michael J. Franklin, Ali Ghodsi, Joseph M. Hellerstein, and Ion Stoica. [Feral Concurrency Control: An Empirical Investigation of Modern Application Integrity](http://www.bailis.org/papers/feral-sigmod2015.pdf). At *ACM International Conference on Management of Data* (SIGMOD), June 2015. [doi:10.1145/2723372.2737784](https://doi.org/10.1145/2723372.2737784)
[^52]: Jaana Dogan. [Things I Wished More Developers Knew About Databases](https://rakyll.medium.com/things-i-wished-more-developers-knew-about-databases-2d0178464f78). *rakyll.medium.com*, April 2020. Archived at [perma.cc/6EFK-P2TD](https://perma.cc/6EFK-P2TD)
[^53]: Michael J. Cahill, Uwe Röhm, and Alan Fekete. [Serializable Isolation for Snapshot Databases](https://www.cs.cornell.edu/~sowell/dbpapers/serializable_isolation.pdf). At *ACM International Conference on Management of Data* (SIGMOD), June 2008. [doi:10.1145/1376616.1376690](https://doi.org/10.1145/1376616.1376690)
[^54]: Dan R. K. Ports and Kevin Grittner. [Serializable Snapshot Isolation in PostgreSQL](https://drkp.net/papers/ssi-vldb12.pdf). At *38th International Conference on Very Large Databases* (VLDB), August 2012.
[^55]: Douglas B. Terry, Marvin M. Theimer, Karin Petersen, Alan J. Demers, Mike J. Spreitzer and Carl H. Hauser. [Managing Update Conflicts in Bayou, a Weakly Connected Replicated Storage System](https://doi.org/10.1145/224056.224070). At *15th ACM Symposium on Operating Systems Principles* (SOSP), December 1995. [doi:10.1145/224056.224070](https://doi.org/10.1145/224056.224070)
[^56]: Hans-Jürgen Schönig. [Constraints over multiple rows in PostgreSQL](https://www.cybertec-postgresql.com/en/postgresql-constraints-over-multiple-rows/). *cybertec-postgresql.com*, June 2021. Archived at [perma.cc/2TGH-XUPZ](https://perma.cc/2TGH-XUPZ)
[^57]: Michael Stonebraker, Samuel Madden, Daniel J. Abadi, Stavros Harizopoulos, Nabil Hachem, and Pat Helland. [The End of an Architectural Era (It’s Time for a Complete Rewrite)](https://vldb.org/conf/2007/papers/industrial/p1150-stonebraker.pdf). At *33rd International Conference on Very Large Data Bases* (VLDB), September 2007.
[^58]: John Hugg. [H-Store/VoltDB Architecture vs. CEP Systems and Newer Streaming Architectures](https://www.youtube.com/watch?v=hD5M4a1UVz8). At *Data @Scale Boston*, November 2014.
[^59]: Robert Kallman, Hideaki Kimura, Jonathan Natkins, Andrew Pavlo, Alexander Rasin, Stanley Zdonik, Evan P. C. Jones, Samuel Madden, Michael Stonebraker, Yang Zhang, John Hugg, and Daniel J. Abadi. [H-Store: A High-Performance, Distributed Main Memory Transaction Processing System](https://www.vldb.org/pvldb/vol1/1454211.pdf). *Proceedings of the VLDB Endowment*, volume 1, issue 2, pages 1496–1499, August 2008.
[^60]: Rich Hickey. [The Architecture of Datomic](https://www.infoq.com/articles/Architecture-Datomic/). *infoq.com*, November 2012. Archived at [perma.cc/5YWU-8XJK](https://perma.cc/5YWU-8XJK)
[^61]: John Hugg. [Debunking Myths About the VoltDB In-Memory Database](https://dzone.com/articles/debunking-myths-about-voltdb). *dzone.com*, May 2014. Archived at [perma.cc/2Z9N-HPKF](https://perma.cc/2Z9N-HPKF)
[^62]: Xinjing Zhou, Viktor Leis, Xiangyao Yu, and Michael Stonebraker. [OLTP Through the Looking Glass 16 Years Later: Communication is the New Bottleneck](https://www.vldb.org/cidrdb/papers/2025/p17-zhou.pdf). At *15th Annual Conference on Innovative Data Systems Research* (CIDR), January 2025.
[^63]: Xinjing Zhou, Xiangyao Yu, Goetz Graefe, and Michael Stonebraker. [Lotus: scalable multi-partition transactions on single-threaded partitioned databases](https://www.vldb.org/pvldb/vol15/p2939-zhou.pdf). *Proceedings of the VLDB Endowment* (PVLDB), volume 15, issue 11, pages 2939–2952, July 2022. [doi:10.14778/3551793.3551843](https://doi.org/10.14778/3551793.3551843)
[^64]: Joseph M. Hellerstein, Michael Stonebraker, and James Hamilton. [Architecture of a Database System](https://dsf.berkeley.edu/papers/fntdb07-architecture.pdf). *Foundations and Trends in Databases*, volume 1, issue 2, pages 141–259, November 2007. [doi:10.1561/1900000002](https://doi.org/10.1561/1900000002)
[^65]: Michael J. Cahill. [Serializable Isolation for Snapshot Databases](https://ses.library.usyd.edu.au/bitstream/handle/2123/5353/michael-cahill-2009-thesis.pdf). PhD Thesis, University of Sydney, July 2009. Archived at [perma.cc/727J-NTMP](https://perma.cc/727J-NTMP)
[^66]: Cristian Diaconu, Craig Freedman, Erik Ismert, Per-Åke Larson, Pravin Mittal, Ryan Stonecipher, Nitin Verma, and Mike Zwilling. [Hekaton: SQL Server’s Memory-Optimized OLTP Engine](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/06/Hekaton-Sigmod2013-final.pdf). At *ACM SIGMOD International Conference on Management of Data* (SIGMOD), pages 1243–1254, June 2013. [doi:10.1145/2463676.2463710](https://doi.org/10.1145/2463676.2463710)
[^67]: Thomas Neumann, Tobias Mühlbauer, and Alfons Kemper. [Fast Serializable Multi-Version Concurrency Control for Main-Memory Database Systems](https://db.in.tum.de/~muehlbau/papers/mvcc.pdf). At *ACM SIGMOD International Conference on Management of Data* (SIGMOD), pages 677–689, May 2015. [doi:10.1145/2723372.2749436](https://doi.org/10.1145/2723372.2749436)
[^68]: D. Z. Badal. [Correctness of Concurrency Control and Implications in Distributed Databases](https://ieeexplore.ieee.org/abstract/document/762563). At *3rd International IEEE Computer Software and Applications Conference* (COMPSAC), November 1979. [doi:10.1109/CMPSAC.1979.762563](https://doi.org/10.1109/CMPSAC.1979.762563)
[^69]: Rakesh Agrawal, Michael J. Carey, and Miron Livny. [Concurrency Control Performance Modeling: Alternatives and Implications](https://people.eecs.berkeley.edu/~brewer/cs262/ConcControl.pdf). *ACM Transactions on Database Systems* (TODS), volume 12, issue 4, pages 609–654, December 1987. [doi:10.1145/32204.32220](https://doi.org/10.1145/32204.32220)
[^70]: Marc Brooker. [Snapshot Isolation vs Serializability](https://brooker.co.za/blog/2024/12/17/occ-and-isolation.html). *brooker.co.za*, December 2024. Archived at [perma.cc/5TRC-CR5G](https://perma.cc/5TRC-CR5G)
[^71]: B. G. Lindsay, P. G. Selinger, C. Galtieri, J. N. Gray, R. A. Lorie, T. G. Price, F. Putzolu, I. L. Traiger, and B. W. Wade. [Notes on Distributed Databases](https://dominoweb.draco.res.ibm.com/reports/RJ2571.pdf). IBM Research, Research Report RJ2571(33471), July 1979. Archived at [perma.cc/EPZ3-MHDD](https://perma.cc/EPZ3-MHDD)
[^72]: C. Mohan, Bruce G. Lindsay, and Ron Obermarck. [Transaction Management in the R\* Distributed Database Management System](https://cs.brown.edu/courses/csci2270/archives/2012/papers/dtxn/p378-mohan.pdf). *ACM Transactions on Database Systems*, volume 11, issue 4, pages 378–396, December 1986. [doi:10.1145/7239.7266](https://doi.org/10.1145/7239.7266)
[^73]: X/Open Company Ltd. [Distributed Transaction Processing: The XA Specification](https://pubs.opengroup.org/onlinepubs/009680699/toc.pdf). Technical Standard XO/CAE/91/300, December 1991. ISBN: 978-1-872-63024-3, archived at [perma.cc/Z96H-29JB](https://perma.cc/Z96H-29JB)
[^74]: Ivan Silva Neto and Francisco Reverbel. [Lessons Learned from Implementing WS-Coordination and WS-AtomicTransaction](https://www.ime.usp.br/~reverbel/papers/icis2008.pdf). At *7th IEEE/ACIS International Conference on Computer and Information Science* (ICIS), May 2008. [doi:10.1109/ICIS.2008.75](https://doi.org/10.1109/ICIS.2008.75)
[^75]: James E. Johnson, David E. Langworthy, Leslie Lamport, and Friedrich H. Vogt. [Formal Specification of a Web Services Protocol](https://www.microsoft.com/en-us/research/publication/formal-specification-of-a-web-services-protocol/). At *1st International Workshop on Web Services and Formal Methods* (WS-FM), February 2004. [doi:10.1016/j.entcs.2004.02.022](https://doi.org/10.1016/j.entcs.2004.02.022)
[^76]: Jim Gray. [The Transaction Concept: Virtues and Limitations](https://jimgray.azurewebsites.net/papers/thetransactionconcept.pdf). At *7th International Conference on Very Large Data Bases* (VLDB), September 1981.
[^77]: Dale Skeen. [Nonblocking Commit Protocols](https://www.cs.utexas.edu/~lorenzo/corsi/cs380d/papers/Ske81.pdf). At *ACM International Conference on Management of Data* (SIGMOD), April 1981. [doi:10.1145/582318.582339](https://doi.org/10.1145/582318.582339)
[^78]: Gregor Hohpe. [Your Coffee Shop Doesn’t Use Two-Phase Commit](https://www.martinfowler.com/ieeeSoftware/coffeeShop.pdf). *IEEE Software*, volume 22, issue 2, pages 64–66, March 2005. [doi:10.1109/MS.2005.52](https://doi.org/10.1109/MS.2005.52)
[^79]: Pat Helland. [Life Beyond Distributed Transactions: An Apostate’s Opinion](https://www.cidrdb.org/cidr2007/papers/cidr07p15.pdf). At *3rd Biennial Conference on Innovative Data Systems Research* (CIDR), January 2007.
[^80]: Jonathan Oliver. [My Beef with MSDTC and Two-Phase Commits](https://blog.jonathanoliver.com/my-beef-with-msdtc-and-two-phase-commits/). *blog.jonathanoliver.com*, April 2011. Archived at [perma.cc/K8HF-Z4EN](https://perma.cc/K8HF-Z4EN)
[^81]: Oren Eini (Ahende Rahien). [The Fallacy of Distributed Transactions](https://ayende.com/blog/167362/the-fallacy-of-distributed-transactions). *ayende.com*, July 2014. Archived at [perma.cc/VB87-2JEF](https://perma.cc/VB87-2JEF)
[^82]: Clemens Vasters. [Transactions in Windows Azure (with Service Bus) – An Email Discussion](https://learn.microsoft.com/en-gb/archive/blogs/clemensv/transactions-in-windows-azure-with-service-bus-an-email-discussion). *learn.microsoft.com*, July 2012. Archived at [perma.cc/4EZ9-5SKW](https://perma.cc/4EZ9-5SKW)
[^83]: Ajmer Dhariwal. [Orphaned MSDTC Transactions (-2 spids)](https://www.eraofdata.com/posts/2008/orphaned-msdtc-transactions-2-spids/). *eraofdata.com*, December 2008. Archived at [perma.cc/YG6F-U34C](https://perma.cc/YG6F-U34C)
[^84]: Paul Randal. [Real World Story of DBCC PAGE Saving the Day](https://www.sqlskills.com/blogs/paul/real-world-story-of-dbcc-page-saving-the-day/). *sqlskills.com*, June 2013. Archived at [perma.cc/2MJN-A5QH](https://perma.cc/2MJN-A5QH)
[^85]: Guozhang Wang, Lei Chen, Ayusman Dikshit, Jason Gustafson, Boyang Chen, Matthias J. Sax, John Roesler, Sophie Blee-Goldman, Bruno Cadonna, Apurva Mehta, Varun Madan, and Jun Rao. [Consistency and Completeness: Rethinking Distributed Stream Processing in Apache Kafka](https://dl.acm.org/doi/pdf/10.1145/3448016.3457556). At *ACM International Conference on Management of Data* (SIGMOD), June 2021. [doi:10.1145/3448016.3457556](https://doi.org/10.1145/3448016.3457556)
