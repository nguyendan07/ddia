---
translationKey: ddia-v2-ch4
title: Lưu trữ và truy xuất
book_kind: chapter
book_number: "4"
book_part: I
weight: 104
breadcrumbs: false
---

<a id="ch_storage"></a>

![](/map/ch03.png)

> *Một trong những nỗi khổ của cuộc sống là ai cũng đặt tên cho sự vật hơi sai một chút. Vì thế mọi thứ trên đời trở nên khó hiểu hơn so với khi chúng được gọi bằng một cái tên khác. Công dụng chính của máy tính không phải là "tính toán" theo nghĩa làm số học. […] Chúng chủ yếu là các hệ thống lưu trữ hồ sơ.*
>
> [Richard Feynman](https://www.youtube.com/watch?v=EKWGGDXe5MA&t=296s),
> *Idiosyncratic Thinking* seminar (1985)

Ở mức cơ bản nhất, một cơ sở dữ liệu cần làm hai việc: khi bạn đưa dữ liệu cho nó, nó phải lưu lại dữ liệu đó, và khi bạn hỏi lại, nó phải trả dữ liệu về cho bạn.

Ở [Chương 3](/ch3#ch_datamodels), ta đã bàn về mô hình dữ liệu và ngôn ngữ truy vấn, tức là định dạng bạn dùng để đưa dữ liệu cho cơ sở dữ liệu và giao diện bạn dùng để lấy dữ liệu về sau này. Trong chương này, ta bàn về cùng vấn đề đó từ góc nhìn của cơ sở dữ liệu: nó lưu dữ liệu ta đưa vào như thế nào, và tìm lại dữ liệu khi ta cần ra sao.

Là lập trình viên ứng dụng, vì sao bạn nên quan tâm đến cách cơ sở dữ liệu lưu trữ và truy xuất dữ liệu ở bên trong? Nhiều khả năng bạn sẽ không tự viết *storage engine* (*storage engine*) từ đầu, nhưng bạn *thực sự* cần chọn ra một storage engine phù hợp với ứng dụng trong số rất nhiều engine hiện có. Và để engine đó chạy tốt trên *tải công việc* của bạn, bạn cần hình dung đại khái nó đang làm gì ở bên dưới.

Cụ thể, có sự khác biệt rất lớn giữa storage engine tối ưu cho tải công việc giao dịch (OLTP) và storage engine tối ưu cho tải công việc phân tích (sự phân biệt này đã được giới thiệu ở [“Hệ thống phân tích và hệ thống tác nghiệp”](/ch1#sec_introduction_analytics)). Chương này bắt đầu bằng hai họ storage engine OLTP: các engine *log-structured* (*log-structured*) ghi ra các file dữ liệu bất biến, và các engine cập nhật tại chỗ như *B-tree*. Cả *key-value store* (*key-value store*) lẫn *chỉ mục phụ* (*secondary index*) đều có thể được xây dựng theo hai họ cấu trúc này.

Sau đó, ở [“Kho lưu trữ dữ liệu phân tích”](/ch4#sec_storage_analytics), ta sẽ bàn về một họ storage engine được tối ưu cho phân tích, còn ở [“Chỉ mục đa chiều (*multidimensional index*) và chỉ mục (*index*) toàn văn”](/ch4#sec_storage_multidimensional), ta sẽ lướt qua các loại chỉ mục (*index*) dùng cho những truy vấn phức tạp hơn như tìm kiếm văn bản.

## Lưu trữ và chỉ mục (*index*) cho hệ thống OLTP {#sec_storage_oltp}

Cơ sở dữ liệu đơn giản nhất thế giới có thể được hiện thực bằng hai hàm Bash:

```bash
#!/bin/bash

db_set () {
  echo "$1,$2" >> database
}

db_get () {
  grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

Hai hàm này hiện thực một key-value store. Gọi `db_set key value` sẽ lưu `key` và `value` vào cơ sở dữ liệu. Khóa và giá trị (gần như) có thể là bất cứ thứ gì bạn muốn, chẳng hạn giá trị có thể là một tài liệu JSON. Sau đó, gọi `db_get key` sẽ tìm giá trị gần nhất gắn với khóa đó và trả về.

Nó chạy được như thế này:

```bash
$ db_set 12 '{"name":"London","attractions":["Big Ben","London Eye"]}'

$ db_set 42 '{"name":"San Francisco","attractions":["Golden Gate Bridge"]}'

$ db_get 42
{"name":"San Francisco","attractions":["Golden Gate Bridge"]}
```

Định dạng lưu trữ bên dưới rất đơn giản: một file văn bản, mỗi dòng chứa một cặp khóa-giá trị phân tách bằng dấu phẩy (gần giống file CSV, nếu bỏ qua vấn đề escape). Mỗi lần gọi `db_set` sẽ nối thêm vào cuối file. Nếu bạn cập nhật một khóa nhiều lần, các phiên bản cũ của giá trị không bị ghi đè — do đó để tìm giá trị mới nhất, bạn phải tìm lần xuất hiện cuối cùng của khóa trong file (vì vậy `db_get` dùng `tail -n 1`):

```bash
$ db_set 42 '{"name":"San Francisco","attractions":["Exploratorium"]}'

$ db_get 42
{"name":"San Francisco","attractions":["Exploratorium"]}

$ cat database
12,{"name":"London","attractions":["Big Ben","London Eye"]}
42,{"name":"San Francisco","attractions":["Golden Gate Bridge"]}
42,{"name":"San Francisco","attractions":["Exploratorium"]}

```

Với một cài đặt đơn giản như vậy, hàm `db_set` có hiệu năng khá tốt, vì việc ghi nối vào cuối file thường rất hiệu quả. Tương tự những gì `db_set` làm, nhiều cơ sở dữ liệu bên trong dùng *log* (*log*), tức là một file dữ liệu chỉ ghi nối (append-only). Cơ sở dữ liệu thực tế còn phải xử lý nhiều vấn đề khác (như ghi đồng thời, thu hồi dung lượng đĩa để log không phình mãi, và xử lý bản ghi mới ghi dở khi khôi phục sau sự cố), nhưng nguyên lý cơ bản là như nhau. Log cực kỳ hữu ích, và ta sẽ còn gặp lại nó nhiều lần trong cuốn sách này.


> [!NOTE]
> Từ *log* thường được dùng để chỉ log của ứng dụng: đoạn văn bản mà ứng dụng xuất ra để mô tả những gì đang diễn ra. Trong cuốn sách này, *log* được dùng theo nghĩa rộng hơn: một chuỗi bản ghi chỉ ghi nối trên đĩa. Người đọc không nhất thiết phải đọc được nó. Nó có thể ở định dạng nhị phân và chỉ dành cho nội bộ hệ thống cơ sở dữ liệu.


Mặt khác, hàm `db_get` có hiệu năng rất tệ nếu cơ sở dữ liệu có nhiều bản ghi. Mỗi lần bạn tra một khóa, `db_get` phải quét toàn bộ file cơ sở dữ liệu từ đầu đến cuối để tìm các lần xuất hiện của khóa. Theo ngôn ngữ thuật toán, chi phí tìm kiếm là *O*(*n*): nếu số bản ghi *n* trong cơ sở dữ liệu tăng gấp đôi thì thời gian tìm kiếm cũng tăng gấp đôi. Như vậy là không tốt.

Để tìm hiệu quả giá trị của một khóa cụ thể trong cơ sở dữ liệu, ta cần một cấu trúc dữ liệu khác: *chỉ mục* (*index*). Chương này sẽ giới thiệu một loạt cấu trúc chỉ mục (*index*) và so sánh chúng. Ý tưởng chung là tổ chức dữ liệu theo một cách nhất định (ví dụ sắp xếp theo một khóa nào đó) để định vị dữ liệu mong muốn nhanh hơn. Nếu muốn tìm kiếm cùng một dữ liệu theo nhiều cách khác nhau, bạn có thể cần nhiều chỉ mục (*index*) khác nhau trên những phần khác nhau của dữ liệu.

Chỉ mục (*index*) là một cấu trúc *bổ sung* được dẫn xuất từ dữ liệu gốc. Nhiều cơ sở dữ liệu cho phép thêm và bớt chỉ mục (*index*) mà không ảnh hưởng đến nội dung cơ sở dữ liệu. Nó chỉ ảnh hưởng đến hiệu năng truy vấn. Việc duy trì cấu trúc bổ sung gây ra chi phí, nhất là khi ghi. Với ghi, khó mà vượt được hiệu năng của việc chỉ ghi nối vào cuối file, vì đó là thao tác ghi đơn giản nhất có thể. Bất kỳ loại chỉ mục (*index*) nào cũng thường làm chậm ghi, vì chỉ mục (*index*) phải được cập nhật mỗi khi dữ liệu được ghi.

Đây là một đánh đổi quan trọng trong hệ thống lưu trữ: chỉ mục (*index*) chọn khéo giúp tăng tốc truy vấn đọc, nhưng mỗi chỉ mục (*index*) tốn thêm dung lượng đĩa và làm chậm ghi, đôi khi đáng kể [^1]. Vì thế, cơ sở dữ liệu thường không tự động đánh chỉ mục (*index*) mọi thứ, mà đòi hỏi người viết ứng dụng hoặc quản trị cơ sở dữ liệu tự chọn chỉ mục (*index*) dựa trên hiểu biết về mẫu truy vấn điển hình của ứng dụng. Nhờ đó bạn chọn được các chỉ mục (*index*) mang lại lợi ích nhiều nhất cho ứng dụng mà không phải gánh chi phí ghi không cần thiết.

### Lưu trữ theo cấu trúc log {#sec_storage_log_structured}

Trước hết, giả sử bạn vẫn muốn lưu dữ liệu trong file chỉ ghi nối do `db_set` tạo ra, và chỉ muốn đọc nhanh hơn. Một chiến lược đánh chỉ mục (*index*) là giữ một hash map trong bộ nhớ, trong đó mỗi khóa ánh xạ tới một byte offset trong file dữ liệu — vị trí của giá trị mới nhất của khóa đó, như minh họa ở {{< xref fig="4-1" page="/ch4" anchor="fig_storage_csv_hash_index" >}}Hình 4-1{{< /xref >}}.

{{< fig num="4-1" id="fig_storage_csv_hash_index" src="/fig/ddia_0401.png" caption="Lưu log các cặp khóa-giá trị theo định dạng giống CSV, được đánh chỉ mục (*index*) bằng một hash map trong bộ nhớ." class="ddia-figure ddia-figure--wide" width="1772" height="872" />}}

Mỗi khi ghi nối một cặp khóa-giá trị mới vào file, bạn cũng cập nhật hash map để nó trỏ tới dữ liệu vừa ghi. Khi tra một giá trị, bạn dùng hash map tìm offset trong file log, seek tới vị trí đó rồi đọc. Nếu phần đó của file dữ liệu đã nằm trong filesystem cache thì việc đọc thậm chí không cần I/O đĩa.

Cách này nhanh hơn nhiều, nhưng vẫn còn một số vấn đề:

* Các mục log cũ đã bị giá trị mới ghi đè vẫn chiếm dung lượng đĩa và không bao giờ được giải phóng. Nếu cứ ghi tiếp vào cơ sở dữ liệu, bạn có thể hết đĩa.
* Hash map không được lưu bền, nên phải dựng lại khi cơ sở dữ liệu khởi động lại. Chẳng hạn, bạn có thể quét toàn bộ file log để tìm byte offset mới nhất của từng khóa. Nếu dữ liệu lớn, việc khởi động lại sẽ rất chậm.
* Hash table phải nằm vừa trong bộ nhớ. Về nguyên tắc có thể duy trì hash table trên đĩa, nhưng tiếc là hash table trên đĩa khó đạt hiệu năng tốt: nó cần nhiều I/O truy cập ngẫu nhiên, tốn kém khi phải mở rộng lúc đầy, và việc xử lý xung đột hash đòi hỏi logic rắc rối [^2].
* Truy vấn khoảng (range query) không hiệu quả. Chẳng hạn, bạn không thể dễ dàng quét mọi khóa từ `10000` đến `19999`. Bạn phải tra từng khóa một trong hash map.

#### Định dạng file SSTable {#the-sstable-file-format}

Trong thực tế, chỉ mục (*index*) cơ sở dữ liệu hiếm khi dùng hash table. Thường gặp hơn là lưu dữ liệu trong một cấu trúc *sắp xếp theo khóa* [^3]. Một ví dụ là *Sorted String Table* (*Sorted String Table*), viết tắt SSTable, như minh họa ở {{< xref fig="4-2" page="/ch4" anchor="fig_storage_sstable_index" >}}Hình 4-2{{< /xref >}}. Định dạng file này cũng lưu các cặp khóa-giá trị, nhưng đảm bảo chúng được sắp xếp theo khóa, và mỗi khóa chỉ xuất hiện một lần trong file.

{{< fig num="4-2" id="fig_storage_sstable_index" src="/fig/ddia_0402.png" caption="SSTable với chỉ mục thưa (*sparse index*), cho phép truy vấn nhảy thẳng tới đúng khối dữ liệu." class="ddia-figure ddia-figure--wide" width="2658" height="1229" />}}

Cách này giúp bạn không phải giữ mọi khóa trong bộ nhớ. Bạn có thể chia các cặp khóa-giá trị trong SSTable thành các *khối* (*block*) vài kilobyte, rồi chỉ lưu khóa đầu tiên của mỗi khối trong chỉ mục (*index*). Loại chỉ mục (*index*) chỉ lưu một số khóa này gọi là *chỉ mục thưa* (*sparse index*). Chỉ mục (*index*) được lưu ở một vùng riêng của SSTable, và có thể dùng B-tree bất biến, trie hoặc bất kỳ cấu trúc dữ liệu nào cho phép tra nhanh một khóa cụ thể [^4].

Chẳng hạn trong {{< xref fig="4-2" page="/ch4" anchor="fig_storage_sstable_index" >}}Hình 4-2{{< /xref >}}, khóa đầu tiên của một khối là `handbag`, còn khóa đầu tiên của khối kế tiếp là `handsome`. Giả sử bạn muốn tìm khóa `handiwork`, vốn không có trong chỉ mục thưa (*sparse index*). Nhờ thứ tự sắp xếp, bạn biết `handiwork` phải nằm giữa `handbag` và `handsome`. Vì vậy bạn seek tới offset của `handbag` rồi quét file từ đó cho đến khi tìm thấy `handiwork`. Nếu quét tới khối kế tiếp mà không thấy thì khóa đó không có trong file. Một khối vài kilobyte có thể quét rất nhanh.

Ngoài ra, mỗi khối bản ghi có thể được nén (phần tô bóng trong {{< xref fig="4-2" page="/ch4" anchor="fig_storage_sstable_index" >}}Hình 4-2{{< /xref >}}). Bên cạnh việc tiết kiệm dung lượng đĩa, nén còn giảm băng thông I/O, với cái giá chỉ là tốn thêm một chút thời gian CPU.

#### Xây dựng và hợp nhất SSTable {#constructing-and-merging-sstables}

Định dạng SSTable tốt cho đọc hơn so với log chỉ ghi nối, nhưng lại làm việc ghi khó khăn hơn. Ta không thể đơn giản ghi nối vào cuối file, vì khi đó file không còn được sắp xếp (trừ khi các khóa tình cờ được ghi theo thứ tự tăng dần). Nếu mỗi lần chèn một khóa vào giữa file lại phải viết lại toàn bộ SSTable thì chi phí ghi sẽ cao đến mức không chấp nhận được.

Giải pháp là dùng cách tiếp cận *log-structured*, kết hợp log chỉ ghi nối với các file đã sắp xếp:

1. Khi nhận một thao tác ghi, thêm nó vào một cấu trúc dữ liệu map có thứ tự trong bộ nhớ, chẳng hạn cây đỏ-đen, skip list [^5] hoặc trie [^6]. Các cấu trúc này cho phép chèn khóa theo thứ tự bất kỳ, tra khóa hiệu quả, và đọc các khóa ra theo thứ tự đã sắp xếp. Cấu trúc dữ liệu trong bộ nhớ này được gọi là *memtable* (*memtable*).
2. Khi memtable vượt quá một ngưỡng nào đó (thường là vài megabyte), ghi nó ra đĩa thành một file SSTable theo thứ tự đã sắp xếp. File SSTable mới này trở thành *segment* (*segment*) mới nhất của cơ sở dữ liệu, được lưu trong file riêng tách biệt với các segment cũ, và mỗi segment có chỉ mục (*index*) riêng. Trong lúc segment mới đang được ghi ra đĩa, cơ sở dữ liệu vẫn có thể tiếp tục ghi vào một instance memtable mới. Khi SSTable ghi xong, bộ nhớ của memtable cũ được giải phóng.
3. Để đọc giá trị của một khóa, trước tiên tìm trong memtable và trong segment mới nhất trên đĩa. Nếu không thấy, tìm tiếp trong segment cũ hơn, và cứ thế cho đến khi tìm thấy khóa hoặc đã duyệt hết segment cũ nhất. Nếu khóa không có trong bất kỳ segment nào thì nó không tồn tại trong cơ sở dữ liệu.
4. Thỉnh thoảng, một tiến trình hợp nhất và compaction chạy nền để gộp các file segment lại và loại bỏ những giá trị đã bị ghi đè hoặc xóa.

Việc hợp nhất các segment tương tự thuật toán *merge sort* [^5], như minh họa ở {{< xref fig="4-3" page="/ch4" anchor="fig_storage_sstable_merging" >}}Hình 4-3{{< /xref >}}. Bạn đọc song song các file đầu vào, nhìn khóa đầu tiên của mỗi file, chép khóa nhỏ nhất theo thứ tự sắp xếp vào file đầu ra, rồi lặp lại. Nếu cùng một khóa xuất hiện trong nhiều file đầu vào, chỉ giữ giá trị mới hơn. Kết quả là một segment mới, cũng được sắp xếp theo khóa và mỗi khóa chỉ có một giá trị. Vì có thể duyệt SSTable từng khóa một nên quá trình này cần rất ít bộ nhớ.

{{< fig num="4-3" id="fig_storage_sstable_merging" src="/fig/ddia_0403.png" caption="Hợp nhất nhiều segment SSTable, chỉ giữ giá trị mới nhất cho mỗi khóa." class="ddia-figure ddia-figure--standard" width="2658" height="1772" />}}

Để tránh mất dữ liệu trong memtable nếu cơ sở dữ liệu gặp sự cố, storage engine còn giữ một log riêng trên đĩa, và mỗi thao tác ghi được ghi nối ngay vào log đó. Log này không được sắp xếp theo khóa, nhưng điều đó không quan trọng, vì mục đích duy nhất của nó là khôi phục memtable sau sự cố. Mỗi khi memtable đã được ghi thành SSTable, phần log tương ứng có thể bị loại bỏ.

Nếu muốn xóa một khóa và giá trị gắn với nó, bạn phải ghi nối vào file dữ liệu một bản ghi xóa đặc biệt gọi là *tombstone* (*tombstone*). Khi các segment log được hợp nhất, tombstone cho tiến trình hợp nhất biết phải loại bỏ mọi giá trị trước đó của khóa này. Khi tombstone đã được hợp nhất vào segment cũ nhất, nó cũng có thể bị xóa.

Thuật toán mô tả ở đây về cơ bản chính là thuật toán được RocksDB [^7], Cassandra, ScyllaDB và HBase [^8] sử dụng. Các hệ thống này đều lấy cảm hứng từ bài báo Bigtable của Google [^9], nơi lần đầu đưa ra các thuật ngữ *SSTable* và *memtable*.

Thuật toán này lần đầu được công bố năm 1996 với tên *Log-Structured Merge-Tree* (*Log-Structured Merge-Tree*), viết tắt *LSM-tree* [^10], xây dựng trên các nghiên cứu trước đó về hệ thống file có cấu trúc log [^11]. Vì thế, các storage engine dựa trên nguyên lý hợp nhất và compaction các file đã sắp xếp thường được gọi là *storage engine LSM*.

Trong storage engine LSM, các file segment được ghi một lần (bằng cách ghi từ memtable ra, hoặc hợp nhất các segment hiện có) và sau đó là bất biến. Việc hợp nhất và compaction segment có thể do thread chạy nền thực hiện. Trong lúc đó, các file segment cũ vẫn tiếp tục phục vụ yêu cầu đọc. Khi hợp nhất xong, các yêu cầu đọc chuyển sang segment đã hợp nhất mới, và các file segment cũ có thể bị xóa.

Các file segment không nhất thiết phải nằm trên đĩa cục bộ. Chúng cũng rất thích hợp để ghi vào kho lưu trữ đối tượng. Chẳng hạn SlateDB và Delta Lake dùng cách này [^12].

Việc file segment bất biến cũng làm khôi phục sau sự cố đơn giản hơn: nếu sự cố xảy ra khi đang ghi memtable hoặc hợp nhất segment, cơ sở dữ liệu chỉ cần xóa SSTable dở dang rồi bắt đầu lại. Log dùng để lưu bền các thao tác ghi vào memtable có thể chứa bản ghi không đầy đủ nếu sự cố xảy ra giữa chừng hoặc đĩa đầy. Thường có thể phát hiện các bản ghi này nhờ checksum trong log, rồi loại bỏ các mục bị hỏng hoặc không đầy đủ. Ta sẽ bàn thêm về tính bền vững và khôi phục sau sự cố ở [Chương 8](/ch8#ch_transactions).

<a id="sec_storage_bloom_filter"></a>

#### Bloom filter {#bloom-filters}

Với LSM storage, việc đọc một khóa đã lâu không được cập nhật, hoặc một khóa hoàn toàn không tồn tại, có thể khá chậm vì storage engine phải kiểm tra nhiều file segment. Để tăng tốc các lần đọc như vậy, các storage engine LSM thường giữ một *Bloom filter* (*Bloom filter*) [^13] cho mỗi segment. Nó cho biết nhanh, ở mức xấp xỉ, một khóa có xuất hiện trong một SSTable hay không.

{{< xref fig="4-4" page="/ch4" anchor="fig_storage_bloom" >}}Hình 4-4{{< /xref >}} minh họa một Bloom filter dài 16 bit chứa hai khóa (thực tế sẽ có nhiều khóa và nhiều bit hơn rất nhiều). Với mỗi khóa trong SSTable, ta tính một hàm hash và nhận được một tập số, rồi diễn giải các số đó thành chỉ số trong mảng bit [^14]. Ta đặt các bit ở những vị trí đó bằng 1 và để các bit còn lại bằng 0. Chẳng hạn, khóa `handbag` có kết quả hash là (2, 9, 4), nên ta đặt các bit 2, 9 và 4 bằng 1. Bitmap thu được được lưu trong SSTable cùng với chỉ mục (*index*) khóa thưa. Nó tốn thêm một chút dung lượng, nhưng Bloom filter thường nhỏ hơn rất nhiều so với phần còn lại của SSTable.

{{< fig num="4-4" id="fig_storage_bloom" src="/fig/ddia_0404.png" caption="Bloom filter cho biết theo xác suất và rất nhanh liệu một khóa có tồn tại trong một SSTable hay không." class="ddia-figure ddia-figure--wide" width="2658" height="1137" />}}

Để kiểm tra một khóa có trong SSTable hay không, ta tính cùng các hash cho khóa đó rồi xem các bit ở những chỉ số tương ứng. Chẳng hạn trong {{< xref fig="4-4" page="/ch4" anchor="fig_storage_bloom" >}}Hình 4-4{{< /xref >}}, ta truy vấn khóa `handheld`, có kết quả hash là (6, 11, 2). Bit 2 bằng 1, còn hai bit kia bằng 0. Các phép toán bit mà mọi CPU đều hỗ trợ cho phép thực hiện các kiểm tra này cực nhanh.

Nếu có ít nhất một bit bằng 0, ta biết chắc khóa không có trong SSTable. Nếu các bit của truy vấn đều bằng 1, khóa rất có thể có trong SSTable, nhưng cũng có thể là các khóa khác tình cờ đã đặt tất cả các bit đó bằng 1. Trường hợp trông như có nhưng thực ra không này gọi là *dương tính giả* (*false positive*).

Xác suất dương tính giả phụ thuộc vào số khóa, số bit đặt cho mỗi khóa và tổng số bit của Bloom filter. Bạn có thể dùng các công cụ tính trực tuyến để chọn tham số phù hợp cho ứng dụng [^15]. Theo kinh nghiệm, cần dành 10 bit Bloom filter cho mỗi khóa trong SSTable để có xác suất dương tính giả khoảng 1%, và cứ thêm 5 bit cho mỗi khóa thì xác suất giảm đi một bậc độ lớn.

Với storage engine LSM, dương tính giả không gây ra vấn đề về tính đúng đắn:

* Nếu Bloom filter nói khóa *không có*, ta có thể yên tâm bỏ qua SSTable đó vì chắc chắn khóa không nằm trong đó.
* Nếu Bloom filter nói khóa *có*, ta phải tra chỉ mục thưa (*sparse index*) và giải mã khối cặp khóa-giá trị để kiểm tra khóa có thật sự ở đó không. Nếu là dương tính giả, ta chỉ tốn thêm chút công vô ích, ngoài ra không có hại gì. Ta tiếp tục tìm trong segment cũ hơn kế tiếp.

#### Chiến lược compaction {#sec_storage_lsm_compaction}

Một chi tiết thiết kế quan trọng của LSM storage là khi nào thực hiện compaction và mỗi lần compaction gồm những SSTable nào. Nhiều hệ thống lưu trữ dựa trên LSM cho phép cấu hình chiến lược compaction. Các lựa chọn phổ biến [^16] [^17] gồm:

Size-tiered compaction
: Dần dần hợp nhất các SSTable mới hơn, nhỏ hơn vào các SSTable cũ hơn, lớn hơn. Các SSTable chứa dữ liệu cũ có thể trở nên rất lớn, và việc hợp nhất chúng cần nhiều dung lượng đĩa tạm. Ưu điểm của chiến lược này là chịu được thông lượng ghi rất cao.

Leveled compaction
: Chia dải khóa thành các SSTable nhỏ hơn và chuyển dữ liệu cũ hơn sang các "tầng" (level) khác nhau. Cách này cho phép compaction diễn ra tăng dần hơn và cần ít dung lượng đĩa hơn size-tiered compaction. Leveled compaction đọc hiệu quả hơn, vì storage engine chỉ cần kiểm tra ít SSTable hơn để xem chúng có chứa khóa cần tìm không.

Theo kinh nghiệm, size-tiered compaction thường hoạt động tốt hơn nếu tải công việc chủ yếu là ghi và ít đọc, còn leveled compaction thường tốt hơn nếu đọc chiếm ưu thế. Leveled compaction cũng có thể có lợi thế nếu một số ít khóa được ghi thường xuyên còn rất nhiều khóa hiếm khi được ghi [^18].

Dù có nhiều điểm tinh tế, ý tưởng cơ bản của LSM-tree — giữ một chuỗi các SSTable được hợp nhất ở chế độ nền — vẫn đơn giản và hiệu quả. Ta sẽ bàn chi tiết hơn về đặc tính hiệu năng của nó ở [“So sánh B-tree và LSM-tree”](/ch4#sec_storage_btree_lsm_comparison).


<a id="sidebar_embedded"></a>

> [!TIP] Storage engine nhúng
>
> Nhiều cơ sở dữ liệu chạy dưới dạng dịch vụ, nhận truy vấn qua mạng, nhưng cũng có những cơ sở dữ liệu *nhúng* (*embedded*), không cung cấp API mạng. Chúng là thư viện chạy trong cùng tiến trình với mã ứng dụng, thường đọc và ghi file trên đĩa cục bộ, và ứng dụng tương tác với chúng bằng các lời gọi hàm thông thường. RocksDB, SQLite, LMDB, DuckDB và KùzuDB đều là storage engine nhúng [^19].
>
> Cơ sở dữ liệu nhúng rất phổ biến trong ứng dụng di động, nơi chúng dùng để lưu dữ liệu cục bộ của người dùng. Ở backend, chúng cũng có thể là lựa chọn phù hợp nếu dữ liệu đủ nhỏ để vừa một máy và không có nhiều giao dịch đồng thời. Chẳng hạn trong hệ thống multitenant, nếu mỗi tenant đủ nhỏ và được cô lập hoàn toàn với nhau (tức là không cần chạy truy vấn kết hợp dữ liệu của nhiều tenant), bạn có thể cân nhắc chạy một instance cơ sở dữ liệu nhúng riêng cho từng tenant [^20].
>
> Các phương pháp lưu trữ và truy xuất bàn trong chương này dùng được cho cả cơ sở dữ liệu nhúng lẫn cơ sở dữ liệu client-server. Ở [Chương 6](/ch6#ch_replication) và [Chương 7](/ch7#ch_sharding), ta sẽ bàn về cách mở rộng cơ sở dữ liệu ra nhiều máy.

### B-tree {#sec_storage_b_trees}

Cách tiếp cận log-structured khá phổ biến, nhưng không phải dạng key-value store duy nhất. Cấu trúc được dùng rộng rãi nhất để đọc và ghi bản ghi cơ sở dữ liệu theo khóa là *B-tree*.

B-tree ra đời năm 1970 [^21] và chưa đầy 10 năm sau đã được gọi là "có mặt khắp nơi" [^22]. Nó đã trụ vững rất tốt theo thời gian. Cho đến nay, nó vẫn là cách hiện thực chỉ mục (*index*) tiêu chuẩn trong hầu như mọi cơ sở dữ liệu quan hệ, và nhiều cơ sở dữ liệu phi quan hệ cũng dùng nó.

Giống SSTable, B-tree lưu các cặp khóa-giá trị đã sắp xếp theo khóa, nhờ đó tra khóa và truy vấn khoảng hiệu quả. Nhưng điểm giống nhau dừng ở đó: B-tree có triết lý thiết kế hoàn toàn khác.

Các chỉ mục (*index*) log-structured vừa xem chia cơ sở dữ liệu thành các *segment* kích thước thay đổi, thường vài megabyte trở lên, được ghi một lần rồi bất biến. Ngược lại, B-tree chia cơ sở dữ liệu thành các *khối* hay *trang* (page) kích thước cố định, và cho phép ghi đè tại chỗ một trang. Kích thước trang truyền thống là 4 KiB, tuy nhiên PostgreSQL hiện mặc định dùng 8 KiB và MySQL mặc định 16 KiB.

Mỗi trang được định danh bằng số trang, nên một trang có thể tham chiếu tới trang khác — tương tự con trỏ, nhưng nằm trên đĩa thay vì trong bộ nhớ. Nếu mọi trang đều nằm trong cùng một file thì số trang nhân với kích thước trang chính là byte offset của trang trong file. Dùng các tham chiếu trang này, ta có thể dựng một cây các trang, như minh họa ở {{< xref fig="4-5" page="/ch4" anchor="fig_storage_b_tree" >}}Hình 4-5{{< /xref >}}.

{{< fig num="4-5" id="fig_storage_b_tree" src="/fig/ddia_0405.png" caption="Tra khóa 251 bằng chỉ mục (*index*) B-tree. Từ trang gốc, đi theo tham chiếu tới trang chứa các khóa 200–300, rồi tới trang chứa các khóa 250–270." class="ddia-figure ddia-figure--wide" width="2658" height="1472" />}}

Một trang được chọn làm *gốc* (*root*) của B-tree. Mỗi khi cần tra một khóa trong chỉ mục (*index*), ta luôn bắt đầu từ đây. Trang gốc chứa một số khóa và các tham chiếu tới trang con. Mỗi trang con phụ trách một dải khóa liên tục, và các khóa nằm giữa các tham chiếu cho biết ranh giới giữa các dải kề nhau. (Cấu trúc này đôi khi được gọi là B+ tree, nhưng ở đây ta không cần phân biệt nó với các biến thể B-tree khác.)

Trong ví dụ ở {{< xref fig="4-5" page="/ch4" anchor="fig_storage_b_tree" >}}Hình 4-5{{< /xref >}}, ta đang tìm khóa 251, nên đi theo tham chiếu trang nằm giữa các ranh giới 200 và 300. Trang tiếp theo có cấu trúc tương tự, chia dải 200–300 thành các dải con nhỏ hơn. Cuối cùng ta tới một *trang lá* (*leaf page*) chứa các khóa riêng lẻ. Trang lá hoặc chứa trực tiếp giá trị của từng khóa, hoặc chứa tham chiếu tới trang nơi lưu giá trị đó.

Số tham chiếu tới trang con trong một trang của B-tree được gọi là *hệ số phân nhánh* (*branching factor*). Chẳng hạn trong {{< xref fig="4-5" page="/ch4" anchor="fig_storage_b_tree" >}}Hình 4-5{{< /xref >}}, hệ số phân nhánh là 6. Trong thực tế, hệ số phân nhánh phụ thuộc vào dung lượng cần để lưu các tham chiếu trang và ranh giới dải, nhưng thường lên tới vài trăm.

Nếu muốn cập nhật giá trị của một khóa đã có trong B-tree, bạn tìm trang lá chứa khóa đó rồi ghi đè trang đó trên đĩa bằng phiên bản chứa giá trị mới. Nếu muốn thêm khóa mới, bạn tìm trang có dải khóa bao trùm khóa mới và thêm khóa vào đó. Nếu trang không đủ chỗ trống cho khóa mới, nó được tách thành hai trang đầy một nửa, và trang cha được cập nhật để phản ánh cách chia dải khóa mới.

{{< fig num="4-6" id="fig_storage_b_tree_split" src="/fig/ddia_0406.png" caption="Làm B-tree lớn thêm bằng cách tách một trang tại khóa biên 337. Trang cha cũng được cập nhật để tham chiếu tới hai trang con." class="ddia-figure ddia-figure--wide" width="2658" height="1588" />}}

Trong {{< xref fig="4-6" page="/ch4" anchor="fig_storage_b_tree_split" >}}Hình 4-6{{< /xref >}}, ta muốn chèn khóa 334, nhưng trang phụ trách dải 333–345 đã đầy. Vì vậy ta tách nó thành hai trang: một trang phụ trách dải 333–337 (gồm cả khóa mới), trang kia phụ trách dải 337–344. Trang cha cũng phải được cập nhật, thêm tham chiếu tới cả hai trang con với 337 làm ranh giới giữa chúng. Nếu trang cha không còn chỗ cho tham chiếu mới, nó cũng bị tách, và việc tách có thể lan ngược lên tận gốc cây. Khi trang gốc bị tách, một gốc mới được tạo phía trên nó. Xóa khóa cũng có thể cần hợp nhất các nút, và phức tạp hơn [^5].

Thuật toán này đảm bảo cây luôn *cân bằng* (*balanced*): một B-tree với *n* khóa luôn có độ sâu *O*(log *n*). Hầu hết cơ sở dữ liệu chỉ cần B-tree sâu ba hoặc bốn tầng, nên bạn không phải lần theo quá nhiều tham chiếu trang để tìm trang cần thiết. (Một cây bốn tầng với trang 4 KiB và hệ số phân nhánh 500 có thể lưu tới 250 TB dữ liệu.)

#### Làm cho B-tree đáng tin cậy {#sec_storage_btree_wal}

Thao tác ghi cơ bản nhất của B-tree là ghi đè một trang trên đĩa bằng dữ liệu mới, với giả định việc ghi đè không làm thay đổi vị trí trang: nghĩa là sau khi một trang bị ghi đè, mọi tham chiếu tới nó vẫn còn hiệu lực. Điều này trái ngược hẳn với các chỉ mục (*index*) log-structured như LSM-tree, vốn chỉ ghi nối vào file (và cuối cùng xóa các file lỗi thời) chứ không bao giờ sửa file tại chỗ.

Ghi đè nhiều trang cùng lúc — chẳng hạn khi tách trang — là thao tác nguy hiểm: nếu cơ sở dữ liệu gặp sự cố khi mới ghi được một phần, kết quả là một cây bị hỏng (chẳng hạn có thể có *orphan page* (*orphan page*), tức trang không thuộc trang cha nào). Nếu phần cứng không thể ghi nguyên tử cả một trang, bạn còn có thể có một trang chỉ được ghi một phần, gọi là *torn page* (*torn page*) [^23].

Để cơ sở dữ liệu có thể khôi phục sau sự cố, các cách hiện thực B-tree thường giữ thêm một cấu trúc dữ liệu trên đĩa: *write-ahead log* (*write-ahead log*, WAL). Đây là một file chỉ ghi nối. Mọi sửa đổi đối với B-tree phải được ghi vào đó trước khi áp dụng lên chính các trang của cây. Khi cơ sở dữ liệu khởi động lại sau sự cố, log này được dùng để đưa B-tree về trạng thái nhất quán [^2] [^24]. Trong hệ thống file, cơ chế tương ứng được gọi là *journaling* (*journaling*).

Để tăng hiệu năng, các cách hiện thực B-tree thường không ghi ngay từng trang đã sửa xuống đĩa mà đệm các trang B-tree trong bộ nhớ một lúc. Khi đó, write-ahead log còn đảm bảo dữ liệu không bị mất khi sự cố xảy ra: miễn là dữ liệu đã được ghi vào WAL và đã được flush xuống đĩa bằng system call `fsync()`, thì nó là bền vững, vì cơ sở dữ liệu có thể khôi phục nó sau sự cố [^25].

#### Các biến thể của B-tree {#b-tree-variants}

Vì B-tree đã tồn tại lâu, nhiều biến thể đã xuất hiện qua nhiều năm. Xin nêu vài ví dụ:

* Một số cơ sở dữ liệu (như LMDB) không ghi đè trang và không dựa vào WAL để khôi phục sau sự cố, mà dùng phương án *copy-on-write* (*copy-on-write*) [^26]. Trang đã sửa được ghi vào vị trí mới, và một phiên bản mới của trang cha trong cây được tạo ra để trỏ tới vị trí mới. Cách này cũng hữu ích cho kiểm soát đồng thời, như ta sẽ thấy ở [“Snapshot isolation và repeatable read”](/ch8#sec_transactions_snapshot_isolation).
* Ta có thể tiết kiệm chỗ trong trang bằng cách không lưu toàn bộ khóa mà chỉ lưu dạng viết tắt. Đặc biệt ở các trang bên trong của cây, khóa chỉ cần chứa đủ thông tin để làm ranh giới giữa các dải khóa. Nhồi được nhiều khóa hơn vào một trang cho phép cây có hệ số phân nhánh cao hơn, và do đó ít tầng hơn.
* Để quét nhanh một dải khóa theo thứ tự sắp xếp, một số cách hiện thực B-tree cố bố trí cây sao cho các trang lá cũng nằm liên tiếp trên đĩa, nhờ đó giảm số lần seek. Tuy nhiên, khi cây lớn dần, rất khó giữ được thứ tự này.
* Ta cũng có thể thêm các con trỏ phụ vào cây. Chẳng hạn, mỗi trang lá có thể tham chiếu tới các trang anh em bên trái và bên phải, cho phép quét các khóa theo thứ tự mà không phải nhảy ngược lên trang cha.

### So sánh B-tree và LSM-tree {#sec_storage_btree_lsm_comparison}

Theo kinh nghiệm, LSM-tree hợp hơn với các ứng dụng nặng về ghi, còn B-tree thường đọc nhanh hơn [^27] [^28]. Tuy nhiên, kết quả benchmark thường rất nhạy với chi tiết của tải công việc. Bạn cần thử nghiệm hệ thống với tải công việc thực tế của mình thì việc so sánh mới có ý nghĩa. Ngoài ra, LSM-tree và B-tree không phải là lựa chọn loại trừ nhau tuyệt đối: storage engine đôi khi kết hợp đặc điểm của cả hai cách, chẳng hạn duy trì nhiều B-tree rồi hợp nhất chúng theo kiểu LSM. Phần này sẽ điểm qua ngắn gọn vài khía cạnh đáng cân nhắc khi đánh giá hiệu năng của storage engine.

#### Hiệu năng đọc {#read-performance}

Tra một khóa trong B-tree cần đọc một trang ở mỗi tầng của cây. Vì số tầng thường nhỏ, việc đọc B-tree nhìn chung nhanh và hiệu năng khá dễ dự đoán. Storage engine LSM thường phải kiểm tra nhiều SSTable ở các giai đoạn compaction khác nhau, nhưng Bloom filter giúp giảm số lần I/O đĩa thực sự phải thực hiện. Cả hai cách đều có thể cho kết quả tốt. Cách nào nhanh hơn tùy vào cách hiện thực storage engine và tải công việc.

B-tree vốn đã có thứ tự nên truy vấn khoảng đơn giản và nhanh. LSM storage cũng tận dụng được việc SSTable được sắp xếp, nhưng phải quét song song mọi segment rồi hợp nhất kết quả. Bloom filter không giúp ích gì cho truy vấn khoảng, vì không thực tế khi tính hash cho mọi khóa tiềm năng trong khoảng. Do đó trong LSM storage, truy vấn khoảng tốn kém hơn truy vấn điểm [^29].

Trong storage engine log-structured, thông lượng ghi cao có thể gây ra các đỉnh độ trễ khi memtable đầy. Điều này xảy ra nếu dữ liệu không thể ghi ra đĩa kịp — có thể vì compaction không theo kịp lượng ghi mới đến. Nhiều storage engine, kể cả RocksDB, áp dụng *backpressure* (*backpressure*) trong tình huống này: chúng tạm dừng mọi đọc và ghi cho đến khi memtable được ghi xong xuống đĩa [^30] [^31].

Về thông lượng đọc, SSD hiện đại (đặc biệt là NVMe) có thể xử lý song song nhiều yêu cầu đọc độc lập. Cả LSM-tree lẫn B-tree đều có thể đạt thông lượng đọc cao, nhưng storage engine phải được thiết kế cẩn thận để tận dụng được khả năng song song này [^32].

#### Ghi tuần tự và ghi ngẫu nhiên {#sidebar_sequential}

Với B-tree, nếu ứng dụng ghi các khóa rải rác khắp không gian khóa, các thao tác đĩa tạo ra cũng rải rác khắp nơi, vì các trang storage engine cần ghi đè có thể nằm ở bất cứ đâu trên đĩa. Ngược lại, storage engine log-structured ghi ra cả một file segment mỗi lần — dù là ghi từ memtable ra hay compaction các segment hiện có — với kích thước lớn hơn nhiều so với một trang trong B-tree.

Kiểu ghi gồm nhiều thao tác nhỏ và nằm rải rác (như trong B-tree) gọi là *ghi ngẫu nhiên* (*random write*), còn kiểu ghi gồm ít thao tác lớn hơn (như trong LSM-tree) gọi là *ghi tuần tự* (*sequential write*). Thông lượng ghi tuần tự của đĩa thường cao hơn ghi ngẫu nhiên, nên trên cùng phần cứng, storage engine log-structured thường chịu được thông lượng ghi cao hơn B-tree. Chênh lệch này đặc biệt rõ trên ổ đĩa quay (HDD). Ngày nay đa số cơ sở dữ liệu dùng ổ thể rắn (SSD) nên khoảng cách đã thu hẹp, nhưng vẫn không thể bỏ qua (xem [“Ghi tuần tự và ghi ngẫu nhiên trên SSD”](/ch4#sidebar_sequential)).

> [!TIP] Ghi tuần tự và ghi ngẫu nhiên trên SSD
>
> Trên ổ đĩa quay (HDD), ghi tuần tự nhanh hơn ghi ngẫu nhiên rất nhiều: ghi ngẫu nhiên phải di chuyển cơ học đầu từ tới vị trí mới và chờ vùng cần ghi trên đĩa quay tới dưới đầu từ, mất đến vài mili giây — theo thang thời gian của máy tính thì như cả một đời người. Ngày nay SSD (ổ thể rắn) đã thay thế HDD trong nhiều trường hợp, kể cả NVMe (Non-Volatile Memory Express, tức bộ nhớ flash nối qua bus PCI Express). Chúng không có giới hạn cơ học này.
>
> Tuy vậy, SSD vẫn có thông lượng ghi tuần tự cao hơn ghi ngẫu nhiên. Bộ nhớ flash có thể đọc và ghi từng trang (thường 4 KiB) nhưng chỉ xóa được theo từng khối (thường 512 KiB). Trong cùng một khối, một số trang có thể vẫn chứa dữ liệu hợp lệ, còn dữ liệu ở các trang khác thì đã vô dụng. Trước khi xóa cả khối, bộ điều khiển phải chuyển các trang chứa dữ liệu hợp lệ sang khối khác. Quá trình này gọi là *garbage collection* (*garbage collection*, GC) [^33].
>
> Ghi tuần tự mỗi lần ghi khối dữ liệu lớn hơn, nên một khối 512 KiB đầy đủ nhiều khả năng thuộc về cùng một file. Khi file đó bị xóa sau này, cả khối có thể được xóa thẳng mà không cần GC. Ngược lại, ghi ngẫu nhiên dễ khiến một khối lẫn lộn trang hợp lệ và trang không hợp lệ, nên GC phải làm nhiều việc chuyển dữ liệu hơn trước khi xóa được khối [^34] [^35] [^36].
>
> Băng thông ghi mà GC chiếm dụng không còn dành cho ứng dụng. Hơn nữa, các thao tác ghi bổ sung do GC sinh ra làm flash mòn nhanh hơn. Vì vậy, ghi ngẫu nhiên làm SSD hao mòn nhanh hơn ghi tuần tự.

#### Khuếch đại ghi {#write-amplification}

Với bất kỳ storage engine nào, một yêu cầu ghi từ ứng dụng sẽ chuyển thành nhiều thao tác I/O trên đĩa bên dưới. Với LSM-tree, một giá trị trước hết được ghi vào log để đảm bảo bền vững, rồi ghi lần nữa khi memtable được ghi xuống đĩa, và lại ghi thêm mỗi lần cặp khóa-giá trị đó tham gia compaction. (Nếu giá trị lớn hơn khóa rất nhiều, có thể giảm chi phí này bằng cách lưu khóa và giá trị riêng, và chỉ compaction các SSTable chứa khóa cùng tham chiếu tới giá trị [^37].)

Chỉ mục (*index*) B-tree cũng phải ghi mỗi mẩu dữ liệu ít nhất hai lần: một lần vào write-ahead log và một lần vào chính trang của cây. Để đảm bảo B-tree khôi phục đúng sau sự cố hoặc mất điện, đôi khi còn phải ghi cả một trang dù chỉ vài byte trong trang thay đổi [^38] [^39].

Tỷ lệ giữa tổng số byte thực sự ghi xuống đĩa cho một tải công việc và số byte cần ghi nếu chỉ ghi vào log chỉ ghi nối, không có chỉ mục (*index*), được gọi là *khuếch đại ghi* (*write amplification*). (Khuếch đại ghi đôi khi cũng được định nghĩa theo số thao tác I/O thay vì số byte.) Trong các ứng dụng nặng về ghi, nút thắt cổ chai có thể là tốc độ cơ sở dữ liệu ghi xuống đĩa. Khi đó, khuếch đại ghi càng cao thì số thao tác ghi mỗi giây xử lý được trong băng thông đĩa có hạn càng ít.

Cả LSM-tree lẫn B-tree đều có khuếch đại ghi. Bên nào tốt hơn phụ thuộc vào nhiều yếu tố, chẳng hạn độ dài khóa và giá trị, hay mức độ thường xuyên ghi đè khóa có sẵn so với chèn khóa mới. Với các tải công việc điển hình, LSM-tree thường có khuếch đại ghi thấp hơn vì không phải ghi cả trang và có thể nén các khối dữ liệu trong SSTable [^40]. Đây cũng là một lý do khiến storage engine LSM phù hợp với tải công việc nặng về ghi.

Ngoài ảnh hưởng đến thông lượng, khuếch đại ghi còn liên quan đến độ hao mòn của SSD: khuếch đại ghi của storage engine càng thấp thì SSD mòn càng chậm.

Khi đo thông lượng ghi của storage engine, thử nghiệm phải chạy đủ lâu để khuếch đại ghi thể hiện ra. Khi bắt đầu ghi vào một LSM-tree rỗng, chưa có compaction nên toàn bộ băng thông đĩa dành cho các thao tác ghi mới. Khi cơ sở dữ liệu lớn dần, các thao tác ghi mới buộc phải chia sẻ băng thông đĩa với compaction.

#### Dung lượng đĩa sử dụng {#disk-space-usage}

B-tree có thể dần trở nên *phân mảnh* (*fragmented*) theo thời gian. Chẳng hạn, sau khi xóa nhiều khóa, file cơ sở dữ liệu có thể còn lại nhiều trang mà B-tree không dùng nữa. Những trang trống này có thể được tái sử dụng khi thêm dữ liệu vào B-tree về sau, nhưng vì chúng nằm giữa file nên khó trả lại cho hệ điều hành, và vẫn chiếm dung lượng hệ thống file. Do đó, cơ sở dữ liệu cần một tiến trình nền để di chuyển và sắp xếp lại các trang, chẳng hạn tiến trình *vacuum* của PostgreSQL [^25].

Phân mảnh ít gây vấn đề hơn với LSM-tree, vì compaction vốn đã định kỳ ghi lại các file dữ liệu, và SSTable không có trang chừa chỗ trống. Ngoài ra, các khối cặp khóa-giá trị trong SSTable dễ nén hơn, nên file đĩa thu được thường nhỏ hơn B-tree. Các khóa và giá trị đã bị ghi đè vẫn chiếm dung lượng cho đến khi được loại bỏ trong compaction, nhưng với leveled compaction thì chi phí này khá thấp [^40] [^41]. Size-tiered compaction (xem [“Chiến lược compaction”](/ch4#sec_storage_lsm_compaction)) dùng nhiều dung lượng đĩa hơn, nhất là dung lượng tạm thời trong lúc compaction.

Nếu bạn cần xóa một số dữ liệu và chắc chắn chúng thực sự biến mất — chẳng hạn để tuân thủ quy định bảo vệ dữ liệu — thì việc có nhiều bản sao cùng tồn tại trên đĩa cũng gây rắc rối. Ở hầu hết storage engine LSM, bản ghi đã xóa vẫn có thể còn ở các tầng cao hơn cho đến khi tombstone đại diện cho thao tác xóa lan qua mọi tầng compaction, và quá trình này có thể kéo dài. Một số thiết kế storage engine chuyên biệt có thể lan truyền thao tác xóa nhanh hơn [^42].

Mặt khác, tính bất biến của các file segment SSTable rất hữu ích khi muốn tạo snapshot tại một thời điểm của cơ sở dữ liệu (chẳng hạn để backup, hoặc sao một bản cơ sở dữ liệu để thử nghiệm): chỉ cần ghi memtable ra rồi ghi nhận hiện có những file segment nào. Miễn là không xóa các file thuộc snapshot, ta hoàn toàn không cần thực sự tạo bản sao của chúng. B-tree ghi đè trang nên khó tạo snapshot hiệu quả như vậy.

### Chỉ mục (*index*) nhiều cột và chỉ mục phụ (*secondary index*) {#sec_storage_index_multicolumn}

Cho đến giờ ta mới chỉ bàn về chỉ mục (*index*) khóa-giá trị, tương tự chỉ mục (*index*) *khóa chính* (*primary key*) trong mô hình quan hệ. Khóa chính định danh duy nhất một hàng trong bảng quan hệ, một tài liệu trong cơ sở dữ liệu tài liệu, hoặc một đỉnh trong cơ sở dữ liệu đồ thị. Các bản ghi khác trong cơ sở dữ liệu có thể tham chiếu tới hàng, tài liệu hay đỉnh đó bằng khóa chính (hoặc ID), và chỉ mục (*index*) dùng để phân giải các tham chiếu này.

*Chỉ mục phụ* (*secondary index*) cũng rất phổ biến. Trong cơ sở dữ liệu quan hệ, bạn có thể tạo nhiều chỉ mục phụ (*secondary index*) trên cùng một bảng bằng lệnh `CREATE INDEX`, cho phép tìm kiếm theo các cột khác ngoài khóa chính. Chẳng hạn, trong {{< xref fig="3-1" page="/ch3" anchor="fig_obama_relational" >}}Hình 3-1{{< /xref >}} (xem [Chương 3](/ch3#ch_datamodels)), nhiều khả năng ta sẽ tạo chỉ mục phụ (*secondary index*) trên cột `user_id` ở các bảng để tìm mọi hàng thuộc cùng một người dùng.

Chỉ mục phụ (*secondary index*) dễ dàng được xây từ chỉ mục (*index*) khóa-giá trị. Điểm khác biệt chính là giá trị được đánh chỉ mục (*index*) trong chỉ mục phụ (*secondary index*) không nhất thiết là duy nhất, tức là có thể có nhiều hàng (hoặc tài liệu, đỉnh) dưới cùng một mục chỉ mục (*index*). Có hai cách giải quyết: biến giá trị trong chỉ mục (*index*) thành danh sách các định danh hàng khớp (như postings list trong chỉ mục (*index*) toàn văn), hoặc làm cho mỗi mục trong chỉ mục (*index*) là duy nhất bằng cách nối thêm định danh hàng vào nó. Cả storage engine cập nhật tại chỗ như B-tree lẫn storage engine log-structured đều có thể dùng để hiện thực chỉ mục phụ (*secondary index*).

#### Lưu giá trị trong chỉ mục (*index*) {#sec_storage_index_heap}

Khóa trong chỉ mục (*index*) là thứ truy vấn tìm kiếm, còn giá trị có thể là một trong nhiều dạng:

* Nếu dữ liệu thực tế (hàng, tài liệu, đỉnh) được lưu trực tiếp trong cấu trúc chỉ mục (*index*) thì đó là *chỉ mục cụm* (*clustered index*). Chẳng hạn, storage engine InnoDB của MySQL luôn dùng khóa chính của bảng làm chỉ mục cụm (*clustered index*), còn SQL Server cho phép chỉ định một chỉ mục cụm (*clustered index*) cho mỗi bảng [^43].
* Cách khác là để giá trị tham chiếu tới dữ liệu thực: có thể là khóa chính của hàng tương ứng (đây là cách chỉ mục phụ (*secondary index*) của InnoDB làm), hoặc tham chiếu trực tiếp tới vị trí trên đĩa. Trong trường hợp sau, nơi lưu các hàng được gọi là *heap file* (*heap file*). Dữ liệu trong heap file không theo thứ tự cụ thể nào, có thể chỉ ghi nối, hoặc ghi nhận các hàng đã xóa để sau này ghi đè bằng dữ liệu mới. Chẳng hạn, Postgres dùng heap file [^44].
* Một giải pháp dung hòa giữa hai cách được gọi là *chỉ mục bao phủ* (*covering index*) hay *chỉ mục có cột bao gồm* (*index with included columns*): hàng đầy đủ vẫn nằm trong heap file hoặc chỉ mục cụm (*clustered index*) theo khóa chính, nhưng chỉ mục (*index*) cũng lưu *một số* cột của bảng [^45]. Nhờ vậy một số truy vấn có thể được trả lời chỉ bằng chỉ mục (*index*) mà không cần phân giải khóa chính hay truy cập heap file. Khi đó ta nói chỉ mục (*index*) *bao phủ* truy vấn. Chỉ mục bao phủ (*covering index*) có thể tăng tốc một số truy vấn, nhưng dữ liệu trùng lặp tốn thêm dung lượng đĩa và làm chậm ghi.

Các chỉ mục (*index*) bàn đến cho tới giờ chỉ ánh xạ một khóa duy nhất tới giá trị. Nếu cần truy vấn đồng thời nhiều cột trong bảng (hoặc nhiều trường trong tài liệu), xem [“Chỉ mục đa chiều (*multidimensional index*) và chỉ mục (*index*) toàn văn”](/ch4#sec_storage_multidimensional).

Khi cập nhật một giá trị mà không đổi khóa, heap file cho phép ghi đè bản ghi tại chỗ, miễn là giá trị mới không lớn hơn giá trị cũ. Nếu giá trị mới lớn hơn thì phức tạp hơn: bản ghi có thể phải chuyển tới một vị trí mới trong heap có đủ chỗ. Khi đó, hoặc mọi chỉ mục (*index*) phải được cập nhật để trỏ tới vị trí mới của bản ghi trong heap, hoặc phải để lại một con trỏ chuyển tiếp ở vị trí cũ [^2].

### Lưu trữ toàn bộ trong bộ nhớ {#sec_storage_inmemory}

Các cấu trúc dữ liệu bàn đến trong chương này đến giờ đều là lời đáp cho những hạn chế của đĩa. So với bộ nhớ chính, đĩa khó xử lý hơn. Với cả đĩa từ lẫn SSD, muốn đọc và ghi tốt thì phải bố trí dữ liệu trên đĩa cẩn thận. Nhưng ta chịu sự bất tiện này vì đĩa có hai ưu điểm đáng kể: nó bền (nội dung không mất khi mất điện) và chi phí trên mỗi GB thấp hơn RAM.

Khi RAM ngày càng rẻ, lợi thế về chi phí trên mỗi GB của đĩa đang giảm dần. Nhiều tập dữ liệu vốn không lớn đến thế, nên hoàn toàn có thể giữ toàn bộ trong bộ nhớ, thậm chí phân tán trên nhiều máy nếu cần. Điều này dẫn đến sự phát triển của *cơ sở dữ liệu trong bộ nhớ* (*in-memory databases*).

Một số key-value store trong bộ nhớ, như Memcached, chỉ dùng để cache, mất dữ liệu khi máy khởi động lại cũng không sao. Nhưng một số cơ sở dữ liệu trong bộ nhớ khác hướng tới tính bền vững, có thể đạt được bằng phần cứng đặc biệt (như RAM chạy bằng pin), bằng cách ghi log các thay đổi xuống đĩa, bằng cách ghi snapshot định kỳ xuống đĩa, hoặc bằng cách sao chép (*replication*) trạng thái trong bộ nhớ sang các máy khác.

Khi cơ sở dữ liệu trong bộ nhớ được khởi động lại, nó cần nạp lại trạng thái từ đĩa hoặc từ một bản sao qua mạng (trừ khi dùng phần cứng đặc biệt). Dù ghi xuống đĩa, nó vẫn là cơ sở dữ liệu trong bộ nhớ, vì đĩa chỉ được dùng làm log chỉ ghi nối để đảm bảo tính bền vững, còn mọi thao tác đọc đều do bộ nhớ phục vụ. Ghi xuống đĩa cũng có lợi ích vận hành: các file trên đĩa có thể được công cụ bên ngoài dễ dàng backup, kiểm tra và phân tích.

Các sản phẩm như VoltDB, SingleStore và Oracle TimesTen là cơ sở dữ liệu trong bộ nhớ dùng mô hình quan hệ. Nhà cung cấp tuyên bố rằng chúng có thể cải thiện hiệu năng mạnh mẽ nhờ loại bỏ toàn bộ chi phí quản lý cấu trúc dữ liệu trên đĩa [^46] [^47]. RAMCloud là key-value store trong bộ nhớ mã nguồn mở có tính bền vững, dùng cách tiếp cận log-structured cho cả dữ liệu trong bộ nhớ lẫn trên đĩa [^48].

Redis và Couchbase cung cấp tính bền vững yếu bằng cách ghi xuống đĩa một cách bất đồng bộ.

Trái với trực giác, ưu thế hiệu năng của cơ sở dữ liệu trong bộ nhớ không đến từ việc không phải đọc đĩa. Nếu đủ bộ nhớ, ngay cả storage engine dựa trên đĩa cũng có thể không bao giờ phải đọc đĩa thật, vì hệ điều hành vốn đã cache các khối đĩa được dùng gần đây trong bộ nhớ. Cơ sở dữ liệu trong bộ nhớ có thể nhanh hơn là vì chúng tránh được chi phí mã hóa các cấu trúc dữ liệu trong bộ nhớ thành dạng có thể ghi xuống đĩa [^49].

Ngoài hiệu năng, một điểm thú vị khác của cơ sở dữ liệu trong bộ nhớ là chúng có thể cung cấp các mô hình dữ liệu khó hiện thực bằng chỉ mục (*index*) trên đĩa. Chẳng hạn, Redis cung cấp giao diện giống cơ sở dữ liệu cho nhiều cấu trúc dữ liệu như hàng đợi ưu tiên và tập hợp. Vì mọi dữ liệu đều nằm trong bộ nhớ nên việc hiện thực tương đối đơn giản.

## Kho lưu trữ dữ liệu phân tích {#sec_storage_analytics}

Kho dữ liệu thường dùng mô hình dữ liệu quan hệ, vì SQL thường rất phù hợp với truy vấn phân tích. Nhiều công cụ phân tích dữ liệu có giao diện đồ họa tạo ra truy vấn SQL, trực quan hóa kết quả và cho phép nhà phân tích khám phá dữ liệu qua các thao tác như *drill-down* (*drill-down*) và *slicing and dicing* (*slicing and dicing*).

Nhìn bề ngoài, kho dữ liệu và cơ sở dữ liệu OLTP quan hệ khá giống nhau vì cả hai đều có giao diện truy vấn SQL. Tuy nhiên, bên trong chúng có thể khác nhau rất xa, vì chúng nhắm tới các mẫu truy vấn hoàn toàn khác nhau. Ngày nay, nhiều nhà cung cấp cơ sở dữ liệu chỉ tập trung vào xử lý giao dịch hoặc vào tải công việc phân tích, chứ không làm cả hai.

Một số cơ sở dữ liệu như Microsoft SQL Server, SAP HANA và SingleStore hỗ trợ cả xử lý giao dịch lẫn kho dữ liệu trong cùng một sản phẩm. Tuy nhiên, các cơ sở dữ liệu xử lý giao dịch/phân tích kết hợp (HTAP) này (đã giới thiệu ở [“Kho dữ liệu”](/ch1#sec_introduction_dwh)) đang ngày càng trở thành hai engine lưu trữ và truy vấn tách biệt, chỉ tình cờ được truy cập qua cùng một giao diện SQL [^50] [^51] [^52] [^53].

### Kho dữ liệu trên đám mây {#sec_cloud_data_warehouses}

Các nhà cung cấp kho dữ liệu như Teradata, Vertica và SAP HANA bán cả kho dữ liệu triển khai tại chỗ theo giấy phép thương mại lẫn giải pháp trên đám mây. Khi ngày càng nhiều khách hàng chuyển lên đám mây, các kho dữ liệu đám mây thế hệ mới như Google Cloud BigQuery, Amazon Redshift và Snowflake đã được áp dụng rộng rãi. Khác với kho dữ liệu truyền thống, kho dữ liệu đám mây tận dụng hạ tầng đám mây có khả năng mở rộng như lưu trữ đối tượng và nền tảng tính toán serverless.

Kho dữ liệu đám mây thường tích hợp tốt hơn với các dịch vụ đám mây khác và linh hoạt hơn. Chẳng hạn, nhiều kho dữ liệu đám mây hỗ trợ tự động nạp log và dễ dàng kết nối với các framework xử lý dữ liệu như Google Cloud Dataflow hay Amazon Web Services Kinesis. Chúng cũng linh hoạt hơn vì tách việc tính toán truy vấn khỏi tầng lưu trữ [^54]. Dữ liệu được lưu bền trong lưu trữ đối tượng thay vì đĩa cục bộ, nên dung lượng lưu trữ và tài nguyên tính toán truy vấn có thể được điều chỉnh riêng rẽ, như đã giới thiệu ở [“Kiến trúc hệ thống cloud-native”](/ch1#sec_introduction_cloud_native).

Các kho dữ liệu mã nguồn mở như Apache Hive, Trino và Apache Spark cũng phát triển cùng với điện toán đám mây. Khi kho lưu trữ dữ liệu phân tích chuyển sang data lake trên lưu trữ đối tượng, các kho dữ liệu mã nguồn mở bắt đầu bị tách rời thành các thành phần độc lập [^55]. Những chức năng trước đây được tích hợp trong một hệ thống duy nhất như Apache Hive nay thường do các thành phần riêng biệt sau đảm nhiệm:

Query engine
: Các query engine như Trino, Apache DataFusion và Presto phân tích cú pháp truy vấn SQL, tối ưu nó thành kế hoạch thực thi rồi thực thi trên dữ liệu. Truy vấn thường cần được xử lý song song bởi các tác vụ phân tán. Một số query engine có sẵn khả năng thực thi tác vụ, số khác dựa vào framework thực thi bên thứ ba như Apache Spark hoặc Apache Flink.

Định dạng lưu trữ
: Định dạng lưu trữ quy định cách mã hóa các hàng của bảng thành byte trong file, thường được lưu trong lưu trữ đối tượng hoặc hệ thống file phân tán [^12]. Query engine có thể truy cập dữ liệu này, và các ứng dụng khác dùng chung data lake cũng vậy. Parquet, ORC, Lance và Nimble đều thuộc loại này, và ta sẽ bàn thêm ở phần sau.

Định dạng bảng
: File được ghi bằng định dạng lưu trữ như Apache Parquet thường là bất biến. Để hỗ trợ chèn và xóa hàng, người ta dùng các định dạng bảng như Apache Iceberg hoặc Databricks Delta. Định dạng bảng dùng một định dạng file để xác định file nào cấu thành một bảng và bảng đó dùng schema nào. Nó cũng có thể cung cấp các tính năng nâng cao như time travel (truy vấn trạng thái của bảng tại một thời điểm trong quá khứ), garbage collection và thậm chí cả giao dịch.

Data catalog
: Giống như định dạng bảng xác định file nào tạo nên một bảng, data catalog xác định bảng nào tạo nên một cơ sở dữ liệu. Catalog dùng để tạo, đổi tên và xóa bảng. Khác với định dạng lưu trữ và định dạng bảng, data catalog như Snowflake Polaris hay Databricks Unity Catalog thường chạy như dịch vụ độc lập và nhận truy vấn qua giao diện REST. Apache Iceberg cũng cung cấp catalog, có thể chạy nhúng trong client hoặc như một tiến trình độc lập. Query engine dùng thông tin trong catalog khi đọc và ghi bảng. Theo truyền thống, catalog được tích hợp với query engine. Việc tách chúng ra giúp các hệ thống khám phá dữ liệu và quản trị dữ liệu (xem [“Hệ thống dữ liệu, pháp luật và xã hội”](/ch1#sec_introduction_compliance)) cũng truy cập được siêu dữ liệu trong catalog.

### Lưu trữ theo cột {#sec_storage_column}

Như đã nêu ở [“Star schema và snowflake schema: các schema cho phân tích”](/ch3#sec_datamodels_analytics), kho dữ liệu thường dùng schema quan hệ, trong đó một bảng fact khổng lồ tham chiếu tới các bảng dimension qua khóa ngoại. Nếu bảng fact có hàng nghìn tỷ hàng và hàng petabyte dữ liệu thì việc lưu trữ và truy vấn hiệu quả trở thành một thách thức lớn. Các bảng dimension thường nhỏ hơn nhiều (chỉ vài triệu hàng), nên phần này sẽ tập trung vào việc lưu trữ bảng fact.

Dù bảng fact thường có hơn 100 cột, một truy vấn kho dữ liệu điển hình mỗi lần chỉ truy cập 4 hoặc 5 cột trong số đó (truy vấn phân tích hiếm khi cần `"SELECT *"`) [^52]. Lấy {{< xref eg="4-1" page="/ch4" anchor="fig_storage_analytics_query" >}}Ví dụ 4-1{{< /xref >}}: nó truy cập rất nhiều hàng (mọi lần bán trái cây tươi hoặc kẹo trong năm 2024) nhưng chỉ cần ba cột của bảng `fact_sales`: `date_key`, `product_sk` và `quantity`. Truy vấn bỏ qua tất cả các cột còn lại.

{{< eg num="4-1" id="fig_storage_analytics_query" caption="Phân tích xem người ta có xu hướng mua trái cây tươi hoặc kẹo vào ngày nào trong tuần" >}}
```sql
SELECT
    dim_date.weekday, dim_product.category,
    SUM(fact_sales.quantity) AS quantity_sold
FROM fact_sales
    JOIN dim_date ON fact_sales.date_key = dim_date.date_key
    JOIN dim_product ON fact_sales.product_sk = dim_product.product_sk
WHERE
    dim_date.year = 2024 AND
    dim_product.category IN ('Fresh fruit', 'Candy')
GROUP BY
    dim_date.weekday, dim_product.category;
```
{{< /eg >}}

Làm thế nào để thực thi truy vấn này hiệu quả?

Hầu hết cơ sở dữ liệu OLTP bố trí lưu trữ theo kiểu *hướng hàng* (*row-oriented*): mọi giá trị của cùng một hàng trong bảng được lưu cạnh nhau. Cơ sở dữ liệu tài liệu cũng tương tự: cả tài liệu thường được lưu thành một chuỗi byte liên tục. Ví dụ CSV ở {{< xref fig="4-1" page="/ch4" anchor="fig_storage_csv_hash_index" >}}Hình 4-1{{< /xref >}} chính là như vậy.

Để xử lý truy vấn như {{< xref eg="4-1" page="/ch4" anchor="fig_storage_analytics_query" >}}Ví dụ 4-1{{< /xref >}}, bạn có thể tạo chỉ mục (*index*) trên `fact_sales.date_key` và/hoặc `fact_sales.product_sk` để cho storage engine biết tìm mọi lần bán hàng của một ngày hoặc một sản phẩm cụ thể ở đâu. Nhưng storage engine hướng hàng vẫn phải nạp các hàng đầy đủ đó (mỗi hàng có hơn 100 thuộc tính) từ đĩa vào bộ nhớ, phân tích cú pháp từng hàng rồi lọc bỏ những hàng không thỏa điều kiện. Việc này có thể mất rất nhiều thời gian.

Ý tưởng đằng sau lưu trữ *hướng cột* (*column-oriented*, hay *columnar*) rất đơn giản: đừng lưu mọi giá trị của một hàng cạnh nhau, mà lưu mọi giá trị của cùng một *cột* cạnh nhau [^56]. Nếu mỗi cột được lưu riêng, truy vấn chỉ cần đọc và phân tích những cột nó dùng, tiết kiệm được rất nhiều việc. {{< xref fig="4-7" page="/ch4" anchor="fig_column_store" >}}Hình 4-7{{< /xref >}} minh họa nguyên lý này bằng một phiên bản mở rộng của bảng fact trong {{< xref fig="3-5" page="/ch3" anchor="fig_dwh_schema" >}}Hình 3-5{{< /xref >}}.


> [!NOTE]
> Lưu trữ theo cột dễ hiểu nhất trong mô hình dữ liệu quan hệ, nhưng cũng áp dụng được cho dữ liệu phi quan hệ. Chẳng hạn, Parquet [^57] là định dạng lưu trữ theo cột hỗ trợ mô hình dữ liệu tài liệu, dựa trên Google Dremel [^58], dùng một kỹ thuật gọi là *shredding* (*shredding*) hay *striping* (*striping*) [^59].


{{< fig num="4-7" id="fig_column_store" src="/fig/ddia_0407.png" caption="Lưu dữ liệu quan hệ theo cột thay vì theo hàng." class="ddia-figure ddia-figure--standard" width="2658" height="1804" />}}

Cách bố trí lưu trữ hướng cột đòi hỏi mỗi cột lưu dữ liệu theo cùng một thứ tự hàng. Do đó, để ghép lại một hàng đầy đủ, bạn lấy mục thứ 23 của từng cột rồi kết hợp chúng thành hàng thứ 23 của bảng.

Trên thực tế, storage engine theo cột không lưu nguyên một cột (có thể có hàng nghìn tỷ hàng) thành một khối liền. Nó chia bảng thành các khối gồm hàng nghìn đến hàng triệu hàng, rồi trong mỗi khối lưu riêng giá trị của từng cột [^60]. Vì nhiều truy vấn chỉ quan tâm một khoảng ngày nhất định nên thường cho mỗi khối chứa các hàng trong một khoảng timestamp nhất định. Khi đó, truy vấn chỉ cần nạp những cột cần dùng từ các khối có khoảng thời gian giao với khoảng ngày cần tìm.

Ngày nay, gần như mọi cơ sở dữ liệu phân tích đều dùng lưu trữ theo cột [^60], từ các kho dữ liệu đám mây lớn như Snowflake [^61], đến cơ sở dữ liệu nhúng đơn node như DuckDB [^62], đến các hệ thống phân tích sản phẩm như Pinot [^63] và Druid [^64]. Các định dạng lưu trữ như Parquet, ORC [^65] [^66], Lance [^67] và Nimble [^68], cùng các định dạng phân tích trong bộ nhớ như Apache Arrow [^65] [^69] và pandas/NumPy [^70], cũng dùng bố cục theo cột. Các cơ sở dữ liệu chuỗi thời gian như InfluxDB IOx [^71] và TimescaleDB [^72] cũng xây dựng trên lưu trữ theo cột.

#### Nén cột {#sec_storage_column_compression}

Ngoài việc chỉ nạp từ đĩa những cột mà truy vấn cần, ta còn có thể nén dữ liệu để giảm hơn nữa yêu cầu về thông lượng đĩa và băng thông mạng. May mắn là lưu trữ theo cột thường rất phù hợp để nén.

Hãy nhìn các chuỗi giá trị của từng cột trong {{< xref fig="4-7" page="/ch4" anchor="fig_column_store" >}}Hình 4-7{{< /xref >}}: chúng trông khá lặp lại, đó là dấu hiệu tốt cho việc nén. Tùy dữ liệu trong cột, có thể dùng các kỹ thuật nén khác nhau. Một kỹ thuật đặc biệt hiệu quả trong kho dữ liệu là *mã hóa bitmap* (*bitmap encoding*), như minh họa ở {{< xref fig="4-8" page="/ch4" anchor="fig_bitmap_index" >}}Hình 4-8{{< /xref >}}.

{{< fig num="4-8" id="fig_bitmap_index" src="/fig/ddia_0408.png" caption="Nén một cột đơn và lưu dưới dạng chỉ mục (*index*) bitmap." class="ddia-figure ddia-figure--standard" width="2658" height="1963" />}}

Thông thường, số giá trị phân biệt trong một cột nhỏ hơn nhiều so với số hàng (chẳng hạn, một nhà bán lẻ có thể có hàng tỷ giao dịch bán hàng nhưng chỉ có 100.000 sản phẩm). Ta có thể biến một cột có *n* giá trị phân biệt thành *n* bitmap riêng biệt: mỗi giá trị phân biệt một bitmap, mỗi hàng một bit. Bit bằng 1 nếu hàng đó mang giá trị này, bằng 0 nếu không.

Một lựa chọn là lưu các bitmap này theo từng bit cho mỗi hàng. Nhưng bitmap thường chứa rất nhiều số 0, tức là rất *thưa*. Khi đó ta có thể dùng run-length encoding: đếm số lượng số 0 hoặc 1 liên tiếp và lưu con số đó, như thể hiện ở phần dưới của {{< xref fig="4-8" page="/ch4" anchor="fig_bitmap_index" >}}Hình 4-8{{< /xref >}}. Roaring bitmap chuyển đổi giữa hai cách biểu diễn bitmap và luôn dùng cách gọn hơn [^73]. Nhờ vậy có thể mã hóa cả một cột cực kỳ hiệu quả.

Chỉ mục (*index*) bitmap như thế rất hợp với các loại truy vấn thường gặp trong kho dữ liệu. Ví dụ:

`WHERE product_sk IN (31, 68, 69):`
: Nạp ba bitmap cho `product_sk = 31`, `product_sk = 68` và `product_sk = 69`, rồi tính phép *OR* theo bit của ba bitmap đó, việc này có thể làm rất hiệu quả.

`WHERE product_sk = 30 AND store_sk = 3:`
: Nạp các bitmap cho `product_sk = 30` và `store_sk = 3`, rồi tính phép *AND* theo bit. Cách này làm được vì thứ tự hàng ở mọi cột đều giống nhau: bit thứ *k* trong bitmap của một cột ứng với cùng hàng với bit thứ *k* trong bitmap của cột khác.

Bitmap cũng có thể trả lời các truy vấn đồ thị, chẳng hạn tìm mọi người dùng trong mạng xã hội được người dùng *X* theo dõi và đồng thời theo dõi người dùng *Y* [^74]. Cơ sở dữ liệu theo cột còn có nhiều phương án nén khác. Xem tài liệu tham khảo [^75].


> [!NOTE]
> Đừng nhầm cơ sở dữ liệu hướng cột với mô hình dữ liệu *wide-column* (*wide-column*, còn gọi là *column-family*), trong đó một hàng có thể có hàng nghìn cột và các hàng không nhất thiết có cùng các cột [^9]. Dù tên gọi giống nhau, cơ sở dữ liệu wide-column thực ra hướng hàng, vì nó lưu mọi giá trị của cùng một hàng cạnh nhau. Google Bigtable, Apache Accumulo và HBase đều thuộc mô hình wide-column.


#### Thứ tự sắp xếp trong lưu trữ theo cột {#sort-order-in-column-storage}

Trong lưu trữ theo cột, thứ tự lưu các hàng không nhất thiết quan trọng. Cách đơn giản nhất là lưu theo thứ tự chèn, vì chèn hàng mới chỉ cần ghi nối vào mỗi cột. Tuy nhiên, ta cũng có thể áp đặt một thứ tự cho dữ liệu, như đã làm với SSTable, và dùng thứ tự đó như một cơ chế chỉ mục (*index*).

Lưu ý rằng sắp xếp từng cột độc lập là vô nghĩa, vì khi đó ta không còn biết các mục nào ở những cột khác nhau thuộc cùng một hàng. Ta chỉ dựng lại được một hàng vì mục thứ *k* của một cột thuộc cùng hàng với mục thứ *k* của cột khác.

Vì vậy, dù dữ liệu được lưu theo cột, việc sắp xếp vẫn phải thực hiện cho cả hàng. Quản trị viên cơ sở dữ liệu có thể chọn các cột để sắp xếp bảng dựa trên hiểu biết về các truy vấn phổ biến. Chẳng hạn, nếu truy vấn thường nhắm vào một khoảng ngày như "tháng trước", có thể chọn `date_key` làm khóa sắp xếp đầu tiên. Khi đó truy vấn chỉ cần quét các hàng của tháng trước, nhanh hơn nhiều so với quét mọi hàng.

Với các hàng có cùng giá trị ở cột sắp xếp đầu tiên, cột thứ hai có thể quyết định tiếp thứ tự. Chẳng hạn, nếu `date_key` là khóa sắp xếp đầu tiên trong {{< xref fig="4-7" page="/ch4" anchor="fig_column_store" >}}Hình 4-7{{< /xref >}}, thì dùng `product_sk` làm khóa sắp xếp thứ hai có thể hữu ích: mọi lần bán cùng một sản phẩm trong cùng một ngày sẽ nằm cạnh nhau trong kho lưu trữ. Điều này giúp các truy vấn nhóm hoặc lọc doanh số theo sản phẩm trong một khoảng ngày nhất định.

Một lợi ích khác của việc sắp xếp là hỗ trợ nén cột. Nếu cột sắp xếp chính không có quá nhiều giá trị phân biệt thì sau khi sắp xếp, ta sẽ có những chuỗi liên tục rất dài trong đó cùng một giá trị lặp đi lặp lại. Dùng run-length encoding đơn giản — như với các bitmap trong {{< xref fig="4-8" page="/ch4" anchor="fig_bitmap_index" >}}Hình 4-8{{< /xref >}} — là có thể nén cột đó xuống còn vài kilobyte, ngay cả khi bảng có hàng tỷ hàng.

Hiệu quả nén mạnh nhất ở khóa sắp xếp đầu tiên. Khóa sắp xếp thứ hai và thứ ba lộn xộn hơn nên không có những chuỗi giá trị lặp dài như vậy. Các cột có thứ tự ưu tiên thấp hơn về cơ bản xuất hiện theo thứ tự ngẫu nhiên, nên có thể nén không tốt. Nhưng chỉ cần sắp xếp được vài cột đầu thì nhìn chung vẫn có lợi.

#### Ghi vào lưu trữ hướng cột {#writing-to-column-oriented-storage}

Ở [“Đặc điểm của xử lý giao dịch và phân tích”](/ch1#sec_introduction_oltp), ta đã thấy rằng các lần đọc trong kho dữ liệu thường tổng hợp một lượng lớn hàng. Lưu trữ theo cột, nén và sắp xếp đều giúp các truy vấn đọc này nhanh hơn. Còn các lần ghi vào kho dữ liệu thường là nạp dữ liệu hàng loạt, thường qua quy trình ETL.

Với lưu trữ theo cột, chèn riêng lẻ một hàng vào giữa bảng đã sắp xếp rất kém hiệu quả, vì mọi cột đã nén đều phải được ghi lại từ vị trí chèn trở đi. Nhưng ghi hàng loạt nhiều hàng cùng lúc phân bổ chi phí ghi lại các cột đó, nên rất hiệu quả.

Việc ghi hàng loạt thường dùng cách tiếp cận log-structured. Mọi thao tác ghi trước tiên đi vào một kho lưu trữ trong bộ nhớ, hướng hàng và có thứ tự. Khi tích lũy đủ lượng ghi, chúng được hợp nhất với các file mã hóa theo cột trên đĩa và ghi thành file mới theo từng lô. Các file cũ giữ nguyên bất biến, còn file mới được ghi một lần, nên lưu trữ đối tượng rất phù hợp để chứa các file này.

Truy vấn phải xem xét cả dữ liệu cột trên đĩa lẫn các thao tác ghi gần đây trong bộ nhớ, rồi kết hợp hai phần. Query execution engine che giấu sự phân biệt này khỏi người dùng. Với nhà phân tích, dữ liệu được chèn, cập nhật hoặc xóa sẽ phản ánh ngay trong các truy vấn sau đó. Snowflake, Vertica, Apache Pinot, Apache Druid và nhiều hệ thống khác làm như vậy [^61] [^63] [^64] [^76].

### Thực thi truy vấn: biên dịch và vector hóa {#sec_storage_vectorized}

Một truy vấn SQL phân tích phức tạp được phân rã thành một *kế hoạch truy vấn* (*query plan*) gồm nhiều giai đoạn thực thi gọi là *toán tử* (*operator*), có thể được phân tán trên nhiều máy để chạy song song. Bộ lập kế hoạch truy vấn có thể tối ưu rất nhiều bằng cách quyết định dùng toán tử nào, theo thứ tự nào, và mỗi toán tử chạy ở đâu.

Bên trong mỗi toán tử, query engine cần thực hiện nhiều thao tác trên các giá trị của một cột, chẳng hạn tìm mọi hàng có giá trị thuộc một tập nào đó (có thể là một phần của phép join), hoặc kiểm tra giá trị có lớn hơn 15 không. Query engine cũng cần xem đồng thời nhiều cột của cùng một hàng, chẳng hạn tìm mọi lần bán hàng mà sản phẩm là chuối và cửa hàng là cửa hàng mục tiêu.

Truy vấn kho dữ liệu phải quét hàng triệu hàng, nên ta phải quan tâm không chỉ lượng dữ liệu đọc từ đĩa mà còn cả thời gian CPU để thực thi các toán tử phức tạp. Toán tử đơn giản nhất giống một trình thông dịch của ngôn ngữ lập trình: với mỗi hàng, nó xem cấu trúc dữ liệu biểu diễn truy vấn để biết cần so sánh hay tính toán gì trên cột nào. Tiếc là cách này quá chậm với nhiều bài toán phân tích. Hai cách thực thi truy vấn hiệu quả đã xuất hiện trong thực tế [^77] như sau:

Biên dịch truy vấn (query compilation)
: Query engine sinh mã thực thi từ truy vấn SQL. Mã này lặp qua từng hàng, đọc giá trị ở các cột liên quan, thực hiện phép so sánh hoặc tính toán cần thiết, và nếu điều kiện thỏa thì chép các giá trị cần thiết vào bộ đệm đầu ra. Sau đó query engine biên dịch mã sinh ra thành mã máy (thường nhờ một trình biên dịch có sẵn như LLVM) và chạy nó trên dữ liệu cột đã mã hóa được nạp vào bộ nhớ. Cách sinh mã này giống biên dịch just-in-time (JIT) mà các runtime như Java Virtual Machine (JVM) sử dụng.

Xử lý vector hóa (vectorized processing)
: Truy vấn vẫn được thông dịch chứ không biên dịch, nhưng để tăng tốc, nó xử lý hàng loạt giá trị của một cột cùng lúc thay vì lặp từng hàng. Cơ sở dữ liệu có sẵn một tập toán tử định nghĩa trước cố định. Ta truyền tham số cho toán tử và nhận về một lô kết quả [^50] [^75].

Chẳng hạn, ta có thể truyền cột `product_sk` và ID của chuối cho toán tử so sánh bằng, và nhận về một bitmap: mỗi giá trị của cột đầu vào ứng với một bit, bằng 1 nếu là chuối. Ta truyền cột `store_sk` và ID của cửa hàng mục tiêu cho cùng toán tử đó để nhận bitmap thứ hai. Cuối cùng, truyền hai bitmap cho toán tử "AND theo bit", như trong {{< xref fig="4-9" page="/ch4" anchor="fig_bitmap_and" >}}Hình 4-9{{< /xref >}}. Bitmap kết quả có một bit 1 cho mỗi lần bán chuối ở cửa hàng cụ thể đó.

{{< fig num="4-9" id="fig_bitmap_and" src="/fig/ddia_0409.png" caption="Phép AND theo bit giữa hai bitmap rất thích hợp cho xử lý vector hóa." class="ddia-figure ddia-figure--wide" width="2658" height="961" />}}

Hai cách này khác nhau nhiều về cách hiện thực, nhưng cả hai đều được dùng trong thực tế [^77]. Cả hai đều khai thác đặc tính của CPU hiện đại để đạt hiệu năng rất tốt:

* ưu tiên truy cập bộ nhớ tuần tự thay vì ngẫu nhiên để giảm cache miss [^78];
* dồn phần lớn công việc vào các vòng lặp trong chặt chẽ (ít lệnh và không có lời gọi hàm), giúp pipeline lệnh của CPU luôn bận và tránh dự đoán rẽ nhánh sai;
* tận dụng song song hóa như đa luồng và lệnh SIMD (single instruction, multiple data) [^79] [^80];
* thao tác trực tiếp trên dữ liệu nén mà không giải mã sang một biểu diễn khác trong bộ nhớ, tránh chi phí cấp phát và copy bộ nhớ.

### Khung nhìn vật chất hóa và data cube {#sec_storage_materialized_views}

Ta đã gặp *khung nhìn vật chất hóa* (*materialized view*) ở [“Vật chất hóa và cập nhật timeline”](/ch2#sec_introduction_materializing). Trong mô hình dữ liệu quan hệ, đó là một đối tượng giống bảng có nội dung là kết quả của một truy vấn nào đó. Khung nhìn vật chất hóa là bản sao kết quả truy vấn được thực sự ghi xuống đĩa, còn khung nhìn ảo chỉ là lối tắt để viết truy vấn. Khi đọc từ khung nhìn ảo, SQL engine sẽ khai triển nó ngay lập tức thành truy vấn bên dưới rồi xử lý truy vấn đã khai triển đó.

Khi dữ liệu bên dưới thay đổi, khung nhìn vật chất hóa cần được cập nhật tương ứng. Một số cơ sở dữ liệu làm việc này tự động, và các hệ thống như Materialize chuyên về việc duy trì khung nhìn vật chất hóa [^81]. Cập nhật khung nhìn làm tăng khối lượng ghi, nhưng khung nhìn vật chất hóa có thể cải thiện hiệu năng đọc nếu tải công việc lặp đi lặp lại cùng các truy vấn.

*Tổng hợp vật chất hóa* (*materialized aggregate*) là một loại khung nhìn vật chất hóa có thể hữu ích trong kho dữ liệu. Như đã nói, truy vấn kho dữ liệu thường dùng các hàm tổng hợp trong SQL như `COUNT`, `SUM`, `AVG`, `MIN` hoặc `MAX`. Nếu nhiều truy vấn dùng cùng một phép tổng hợp, việc xử lý lại dữ liệu thô mỗi lần là lãng phí — sao không cache lại các giá trị count hay sum được dùng nhiều nhất? *Data cube* (*data cube*, hay *OLAP cube*) tạo một lưới các phép tổng hợp được nhóm theo các chiều khác nhau, nhằm đúng mục đích cache đó [^82]. {{< xref fig="4-10" page="/ch4" anchor="fig_data_cube" >}}Hình 4-10{{< /xref >}} cho một ví dụ.

{{< fig num="4-10" id="fig_data_cube" src="/fig/ddia_0410.png" caption="Hai chiều của một data cube, tổng hợp dữ liệu bằng phép tính tổng." class="ddia-figure ddia-figure--wide" width="2880" height="1484" />}}

Trước hết hãy giả sử mỗi bản ghi fact chỉ có khóa ngoại tham chiếu tới hai bảng dimension — trong {{< xref fig="4-10" page="/ch4" anchor="fig_data_cube" >}}Hình 4-10{{< /xref >}} là `date_key` và `product_sk`. Khi đó có thể vẽ một bảng hai chiều, một trục là ngày và trục kia là sản phẩm. Mỗi ô chứa giá trị tổng hợp (ví dụ `SUM`) của một thuộc tính (ví dụ `net_price`) trên mọi bản ghi fact có tổ hợp ngày–sản phẩm tương ứng. Sau đó có thể áp dụng cùng phép tổng hợp dọc theo từng hàng hoặc từng cột để được bản tóm tắt giảm đi một chiều: doanh số theo sản phẩm bất kể ngày, hoặc doanh số theo ngày bất kể sản phẩm.

Nhìn chung, fact thường có nhiều hơn hai chiều. Trong {{< xref fig="3-5" page="/ch3" anchor="fig_dwh_schema" >}}Hình 3-5{{< /xref >}} có năm chiều: ngày, sản phẩm, cửa hàng, khuyến mãi và khách hàng. Khó hình dung một siêu khối năm chiều, nhưng nguyên lý vẫn như vậy: mỗi ô chứa doanh số của một tổ hợp ngày–sản phẩm–cửa hàng–khuyến mãi–khách hàng cụ thể, và các giá trị này sau đó có thể được tóm tắt lặp lại dọc theo từng chiều.

Lợi ích của data cube vật chất hóa là một số truy vấn trở nên rất nhanh, vì kết quả thực chất đã được tính sẵn. Chẳng hạn, nếu muốn biết tổng doanh số của từng cửa hàng hôm qua, bạn chỉ cần xem tổng ở chiều tương ứng, không phải quét hàng triệu hàng.

Nhược điểm là data cube không linh hoạt bằng truy vấn dữ liệu thô. Chẳng hạn, không thể tính tỷ lệ doanh số đến từ các mặt hàng giá trên 100 đô-la nếu giá không phải là một trong các chiều. Vì vậy, hầu hết kho dữ liệu cố giữ càng nhiều dữ liệu thô càng tốt, và chỉ dùng các phép tổng hợp như data cube làm cách tối ưu hiệu năng cho một số truy vấn nhất định.


## Chỉ mục đa chiều (*multidimensional index*) và chỉ mục (*index*) toàn văn {#sec_storage_multidimensional}

B-tree và LSM-tree ở nửa đầu chương này cho phép truy vấn khoảng trên một thuộc tính duy nhất. Chẳng hạn, nếu khóa là tên người dùng, chúng cho phép đánh chỉ mục (*index*) để tìm hiệu quả mọi tên bắt đầu bằng chữ L. Nhưng đôi khi chỉ tìm theo một thuộc tính là chưa đủ.

Loại chỉ mục (*index*) nhiều cột phổ biến nhất gọi là *chỉ mục ghép* (*concatenated index*). Nó kết hợp nhiều trường thành một khóa bằng cách nối cột này sau cột kia. Thứ tự nối do định nghĩa chỉ mục (*index*) quy định. Điều này giống chỉ mục (*index*) của danh bạ điện thoại giấy kiểu cũ, ánh xạ từ (*họ*, *tên*) sang số điện thoại. Vì chỉ mục (*index*) được sắp xếp theo thứ tự này, ta có thể dùng nó để tìm mọi người có một họ nhất định, hoặc mọi người có một tổ hợp *họ–tên* cụ thể. Nhưng chỉ mục (*index*) này vô dụng nếu bạn muốn tìm mọi người có một tên nhất định.

*Chỉ mục đa chiều* (*multidimensional index*) cho phép truy vấn đồng thời nhiều cột, điều đặc biệt quan trọng với dữ liệu không gian địa lý. Chẳng hạn, cơ sở dữ liệu của một trang web tìm nhà hàng có thể lưu vĩ độ và kinh độ của từng nhà hàng. Khi người dùng xem bản đồ, trang web cần tìm mọi nhà hàng nằm trong vùng hình chữ nhật mà bản đồ đang hiển thị. Việc này đòi hỏi một truy vấn khoảng hai chiều như sau:

```sql
SELECT * FROM restaurants WHERE latitude > 51.4946 AND latitude < 51.5079
    AND longitude > -0.1162 AND longitude < -0.1004;
```

Chỉ mục ghép (*concatenated index*) trên các cột vĩ độ và kinh độ không thể trả lời hiệu quả truy vấn này: nó chỉ có thể trả về mọi nhà hàng trong một khoảng vĩ độ nào đó (ở mọi kinh độ), hoặc mọi nhà hàng trong một khoảng kinh độ nào đó (ở bất kỳ vĩ độ nào giữa hai cực Bắc và Nam), chứ không thể ràng buộc cả hai cùng lúc.

Một cách là dùng đường cong lấp đầy không gian (space-filling curve) để chuyển vị trí hai chiều thành một con số duy nhất, rồi dùng chỉ mục (*index*) B-tree thông thường [^83]. Phổ biến hơn là dùng các chỉ mục (*index*) không gian chuyên biệt như R-tree hay Bkd-tree [^84]. Chúng chia nhỏ không gian sao cho các điểm dữ liệu gần nhau có xu hướng nằm trong cùng một cây con. Chẳng hạn, PostGIS hiện thực chỉ mục (*index*) không gian địa lý dưới dạng R-tree bằng cơ chế chỉ mục (*index*) Generalized Search Tree của PostgreSQL [^85]. Cũng có thể dùng lưới đều gồm tam giác, hình vuông hoặc lục giác [^86].

Chỉ mục đa chiều (*multidimensional index*) không chỉ dùng cho vị trí địa lý. Chẳng hạn, trên trang thương mại điện tử, bạn có thể dùng chỉ mục (*index*) ba chiều trên (*đỏ*, *xanh lục*, *xanh lam*) để tìm sản phẩm trong một khoảng màu nhất định, hoặc trong cơ sở dữ liệu quan sát thời tiết, bạn có thể dùng chỉ mục (*index*) hai chiều trên (*ngày*, *nhiệt độ*) để tìm hiệu quả mọi quan sát trong năm 2013 có nhiệt độ từ 25 đến 30℃. Với chỉ mục (*index*) một chiều, bạn sẽ phải hoặc quét mọi bản ghi năm 2013 (bất kể nhiệt độ) rồi lọc theo nhiệt độ, hoặc làm ngược lại. Chỉ mục (*index*) hai chiều có thể thu hẹp kết quả đồng thời theo timestamp và nhiệt độ [^87].

### Tìm kiếm toàn văn {#sec_storage_full_text}

*Tìm kiếm toàn văn* (*full-text search*) cho phép tìm một tập tài liệu văn bản (trang web, mô tả sản phẩm, v.v.) theo từ khóa, trong đó từ khóa có thể xuất hiện ở bất kỳ đâu trong văn bản [^88]. Truy xuất thông tin là một lĩnh vực lớn và chuyên sâu, thường còn liên quan đến xử lý riêng cho từng ngôn ngữ. Chẳng hạn, một số ngôn ngữ châu Á được viết không có khoảng trắng hay dấu câu giữa các từ, nên để tách văn bản thành từ cần một mô hình cho biết chuỗi ký tự nào tạo thành một từ. Tìm kiếm toàn văn cũng thường cần khớp các từ giống nhưng không hoàn toàn trùng (như lỗi chính tả hoặc các dạng ngữ pháp khác nhau của cùng một từ) và các từ đồng nghĩa. Những vấn đề này nằm ngoài phạm vi cuốn sách.

Tuy nhiên, về bản chất, tìm kiếm toàn văn cũng có thể xem là một loại truy vấn đa chiều: mỗi từ có thể xuất hiện trong văn bản (một *term*) là một chiều. Tài liệu chứa term *x* có giá trị 1 ở chiều *x*, còn tài liệu không chứa *x* có giá trị 0. Tìm các tài liệu nhắc tới "táo đỏ" nghĩa là tìm tài liệu có giá trị 1 ở cả chiều *đỏ* lẫn chiều *táo*. Do đó số chiều có thể cực lớn.

Nhiều công cụ tìm kiếm dùng *chỉ mục đảo ngược* (*inverted index*) để trả lời loại truy vấn này. Đó là một cấu trúc khóa-giá trị, trong đó khóa là term và giá trị là danh sách ID của mọi tài liệu chứa term đó — *postings list* (*postings list*). Nếu ID tài liệu là các số liên tiếp thì postings list cũng có thể biểu diễn thành bitmap thưa như {{< xref fig="4-8" page="/ch4" anchor="fig_bitmap_index" >}}Hình 4-8{{< /xref >}}: bit thứ *n* trong bitmap của term *x* bằng 1 nếu tài liệu có ID *n* chứa term *x* [^89].

Việc tìm mọi tài liệu chứa cả term *x* lẫn *y* khi đó giống truy vấn kho dữ liệu vector hóa tìm các hàng thỏa hai điều kiện ({{< xref fig="4-9" page="/ch4" anchor="fig_bitmap_and" >}}Hình 4-9{{< /xref >}}): nạp bitmap của *x* và *y* rồi tính AND theo bit. Dù bitmap được mã hóa run-length, việc này vẫn thực hiện hiệu quả.

Lucene, engine chỉ mục (*index*) toàn văn mà Elasticsearch và Solr dùng, hoạt động theo cách này [^90]. Nó lưu ánh xạ từ term đến postings list trong các file có thứ tự giống SSTable, và hợp nhất các file đó ở chế độ nền bằng cách tiếp cận log-structured đã nêu ở phần trước của chương [^91]. Chỉ mục (*index*) GIN của PostgreSQL cũng dùng postings list để hỗ trợ tìm kiếm toàn văn và đánh chỉ mục (*index*) bên trong tài liệu JSON [^92] [^93].

Thay vì tách văn bản thành từ, một lựa chọn khác là tìm mọi chuỗi con có độ dài *n*, gọi là *n*-gram. Chẳng hạn, các 3-gram (*n* = 3) của chuỗi `"hello"` là `"hel"`, `"ell"` và `"llo"`. Nếu đánh chỉ mục đảo ngược (*inverted index*) cho mọi 3-gram, ta có thể tìm kiếm mọi chuỗi con dài ít nhất ba ký tự. Chỉ mục (*index*) như vậy thậm chí cho phép dùng biểu thức chính quy trong truy vấn tìm kiếm, với cái giá là kích thước khá lớn [^94].

Để xử lý lỗi chính tả trong tài liệu hoặc truy vấn, Lucene có thể tìm các từ nằm trong một khoảng cách chỉnh sửa (edit distance) nhất định so với từ cần tìm (khoảng cách chỉnh sửa 1 nghĩa là đã thêm, xóa hoặc thay một chữ cái) [^95]. Để làm vậy, nó lưu tập term dưới dạng một automaton hữu hạn trạng thái trên các ký tự của khóa, tương tự *trie* (*trie*) [^96], và chuyển nó thành *Levenshtein automaton* (*Levenshtein automaton*), cho phép tìm hiệu quả các từ trong một khoảng cách chỉnh sửa cho trước [^97].


### Vector embedding {#id92}

Tìm kiếm ngữ nghĩa không chỉ xử lý từ đồng nghĩa và lỗi chính tả, mà còn cố hiểu khái niệm mà tài liệu diễn đạt và ý định của người dùng. Chẳng hạn, nếu một trang trong trung tâm trợ giúp có tiêu đề "Hủy đăng ký", người dùng tìm "cách đóng tài khoản" hay "chấm dứt hợp đồng" cũng nên tìm thấy trang đó: các cách nói này hoàn toàn khác từ ngữ nhưng ý nghĩa rất gần nhau.

Để hiểu ngữ nghĩa của tài liệu, tức ý nghĩa mà nó biểu đạt, chỉ mục tìm kiếm (*search index*) ngữ nghĩa dùng mô hình embedding để chuyển tài liệu thành một vector các số thực dấu phẩy động, gọi là *vector embedding* (*vector embedding*). Vector này biểu diễn một điểm trong không gian đa chiều, và mỗi số thực biểu diễn vị trí của tài liệu trên một trục chiều. Nếu các tài liệu đầu vào có ngữ nghĩa tương tự, mô hình embedding sẽ sinh ra các vector nằm gần nhau trong không gian đa chiều.


> [!NOTE]
> Ta đã gặp thuật ngữ *xử lý vector hóa* ở [“Thực thi truy vấn: biên dịch và vector hóa”](/ch4#sec_storage_vectorized). "Vector" trong tìm kiếm ngữ nghĩa mang nghĩa khác: trong xử lý vector hóa, vector là một lô bit có thể xử lý bằng mã được tối ưu chuyên biệt, còn trong mô hình embedding, vector là một danh sách số thực dấu phẩy động biểu diễn một vị trí trong không gian đa chiều.


Chẳng hạn, vector embedding ba chiều của một trang Wikipedia về nông nghiệp có thể là `[0.1, 0.22, 0.11]`. Một trang về rau củ sẽ gần nó, có thể có vector `[0.13, 0.19, 0.24]`. Một trang về star schema có thể có embedding `[0.82, 0.39, -0.74]`, tương đối xa. Chỉ nhìn các con số cũng thấy hai vector đầu gần nhau hơn vector thứ ba.

Mô hình embedding thực tế dùng vector lớn hơn nhiều, thường có hơn 1.000 số, nhưng nguyên lý giống nhau. Ta không cố hiểu từng con số đại diện cho điều gì. Chúng chỉ là cách mô hình embedding trỏ tới một vị trí trong không gian đa chiều trừu tượng. Công cụ tìm kiếm dùng các hàm khoảng cách như cosine similarity hay khoảng cách Euclid để đo khoảng cách giữa các vector. Cosine similarity tính cosine của góc giữa hai vector để xem chúng gần nhau đến đâu, còn khoảng cách Euclid tính khoảng cách đường thẳng giữa hai điểm trong không gian.

Nhiều mô hình embedding ban đầu như Word2Vec [^98], BERT [^99] và GPT [^100] xử lý dữ liệu văn bản, thường được hiện thực bằng mạng nơ-ron. Về sau, các nhà nghiên cứu cũng tạo ra mô hình embedding cho video, âm thanh và hình ảnh. Gần đây, kiến trúc mô hình còn tiến tới *đa phương thức* (*multimodal*): một mô hình duy nhất có thể sinh vector embedding cho nhiều phương thức như văn bản và hình ảnh.

Khi người dùng nhập truy vấn, công cụ tìm kiếm ngữ nghĩa đưa truy vấn và ngữ cảnh liên quan (chẳng hạn vị trí người dùng) cho mô hình embedding để sinh vector embedding của truy vấn. Sau đó, công cụ tìm kiếm phải dùng chỉ mục (*index*) vector để tìm các tài liệu có vector embedding tương tự truy vấn.

Chỉ mục (*index*) vector lưu vector embedding của một tập tài liệu. Khi truy vấn, ta đưa vector embedding của chính truy vấn vào, và chỉ mục (*index*) trả về các tài liệu có vector gần nhất với vector truy vấn. R-tree đã nói ở trên không phù hợp với vector có số chiều rất cao, nên cần các chỉ mục (*index*) vector chuyên biệt, chẳng hạn:

Flat index
: Vector được lưu nguyên trong chỉ mục (*index*). Truy vấn phải đọc từng vector và đo khoảng cách giữa nó với vector truy vấn. Flat index cho kết quả chính xác, nhưng việc đo khoảng cách giữa truy vấn và từng vector rất chậm.

Inverted file (IVF) index
: Gom không gian vector thành các phân vùng vector, gọi là *tâm cụm* (*centroid*), để giảm số vector phải so sánh. Chỉ mục (*index*) IVF nhanh hơn flat index nhưng chỉ cho kết quả xấp xỉ: vector truy vấn và vector tài liệu có thể rất gần nhau nhưng lại rơi vào hai phân vùng khác nhau. Khi truy vấn chỉ mục (*index*) IVF, trước hết phải chỉ định số *probe*, tức số phân vùng cần kiểm tra. Số probe càng lớn thì truy vấn càng chính xác nhưng càng chậm, vì phải so sánh nhiều vector hơn.

Hierarchical Navigable Small World (HNSW)
: Chỉ mục (*index*) HNSW duy trì nhiều tầng của không gian vector, như minh họa ở {{< xref fig="4-11" page="/ch4" anchor="fig_vector_hnsw" >}}Hình 4-11{{< /xref >}}. Mỗi tầng được biểu diễn thành một đồ thị, trong đó nút đại diện cho vector và cạnh cho biết các vector gần nhau. Truy vấn bắt đầu bằng việc tìm vector gần nhất ở tầng trên cùng, nơi chỉ có rất ít nút, rồi đi xuống cùng nút đó ở tầng kế tiếp, nơi các kết nối dày đặc hơn, và đi theo các cạnh để tìm vector gần vector truy vấn hơn. Quá trình lặp lại cho đến tầng dưới cùng. Giống IVF, HNSW cũng là chỉ mục (*index*) xấp xỉ.

{{< fig num="4-11" id="fig_vector_hnsw" src="/fig/ddia_0411.png" caption="Tìm trong chỉ mục (*index*) HNSW mục cơ sở dữ liệu gần nhất với một vector truy vấn cho trước." class="ddia-figure ddia-figure--wide" width="2658" height="1452" />}}


Nhiều cơ sở dữ liệu vector phổ biến hiện thực chỉ mục (*index*) IVF và HNSW. Thư viện Faiss của Facebook cung cấp nhiều biến thể của cả hai [^101], và pgvector của PostgreSQL cũng hỗ trợ cả hai loại chỉ mục (*index*) [^102]. Chi tiết đầy đủ của thuật toán IVF và HNSW nằm ngoài phạm vi cuốn sách, nhưng các bài báo giới thiệu chúng là nguồn tham khảo tốt [^103] [^104].

## Tóm tắt {#summary}

Trong chương này, ta đã cố đi sâu vào cách cơ sở dữ liệu xử lý việc lưu trữ và truy xuất. Điều gì xảy ra khi bạn lưu dữ liệu vào cơ sở dữ liệu, và điều gì xảy ra khi bạn truy vấn lại dữ liệu đó sau này?

[“Hệ thống phân tích và hệ thống tác nghiệp”](/ch1#sec_introduction_analytics) đã giới thiệu sự phân biệt giữa xử lý giao dịch (OLTP) và phân tích (OLAP). Chương này cho thấy storage engine tối ưu cho OLTP rất khác với storage engine tối ưu cho phân tích:

* Hệ thống OLTP được tối ưu cho lượng lớn yêu cầu, mỗi yêu cầu chỉ đọc và ghi một số ít bản ghi và cần phản hồi nhanh. Các bản ghi thường được truy cập qua khóa chính hoặc chỉ mục phụ (*secondary index*). Các chỉ mục (*index*) này thường là ánh xạ có thứ tự từ khóa sang bản ghi, và cũng hỗ trợ truy vấn khoảng.
* Hệ thống phân tích như kho dữ liệu được tối ưu cho các truy vấn đọc phức tạp phải quét một lượng lớn bản ghi. Chúng thường dùng bố cục lưu trữ theo cột có nén để giảm tối đa lượng dữ liệu truy vấn phải đọc từ đĩa, và dùng biên dịch just-in-time hoặc vector hóa truy vấn để giảm tối đa thời gian CPU xử lý dữ liệu.

Về phía OLTP, ta đã thấy storage engine thuộc hai trường phái tư tưởng chính:

* Trường phái log-structured chỉ cho phép ghi nối vào file và xóa các file lỗi thời, không bao giờ cập nhật file đã ghi. SSTable, LSM-tree, RocksDB, Cassandra, HBase, ScyllaDB, Lucene và các hệ thống khác đều thuộc loại này. Nói chung, storage engine log-structured cung cấp thông lượng ghi cao.
* Trường phái cập nhật tại chỗ coi đĩa là một tập các trang kích thước cố định có thể ghi đè. B-tree là ví dụ tiêu biểu nhất của triết lý này, được dùng trong mọi cơ sở dữ liệu quan hệ OLTP lớn và nhiều cơ sở dữ liệu phi quan hệ. Theo kinh nghiệm, B-tree thường hợp với đọc hơn, có thông lượng đọc cao hơn và thời gian phản hồi ngắn hơn so với lưu trữ log-structured.

Sau đó, ta xem xét các chỉ mục (*index*) có thể tìm kiếm đồng thời nhiều điều kiện: chỉ mục đa chiều (*multidimensional index*) như R-tree có thể tìm điểm trên bản đồ theo cả kinh độ lẫn vĩ độ, còn chỉ mục tìm kiếm (*search index*) toàn văn có thể tìm nhiều từ khóa xuất hiện trong cùng một đoạn văn bản. Cuối cùng, cơ sở dữ liệu vector dùng để tìm kiếm ngữ nghĩa trên tài liệu văn bản và các phương tiện khác. Chúng biểu diễn dữ liệu thành các vector nhiều chiều và tìm tài liệu tương tự bằng cách so sánh độ tương đồng giữa các vector.

Là lập trình viên ứng dụng, nếu nắm được cơ chế bên trong của storage engine, bạn sẽ đánh giá tốt hơn công cụ nào phù hợp nhất với ứng dụng của mình. Khi cần điều chỉnh các tham số tinh chỉnh của cơ sở dữ liệu, hiểu biết này cũng giúp bạn hình dung việc tăng hay giảm một tham số sẽ có tác động ra sao.

Dù chương này không thể biến bạn thành chuyên gia tinh chỉnh một storage engine cụ thể, hy vọng nó đã cung cấp đủ khái niệm và từ vựng để bạn đọc hiểu tài liệu của cơ sở dữ liệu mình chọn.


### Tài liệu tham khảo

[^1]: Nikolay Samokhvalov. [How partial, covering, and multicolumn indexes may slow down UPDATEs in PostgreSQL](https://postgres.ai/blog/20211029-how-partial-and-covering-indexes-affect-update-performance-in-postgresql). *postgres.ai*, October 2021. Archived at [perma.cc/PBK3-F4G9](https://perma.cc/PBK3-F4G9)
[^2]: Goetz Graefe. [Modern B-Tree Techniques](https://doi.org/10.1561/1900000028). *Foundations and Trends in Databases*, volume 3, issue 4, pages 203–402, August 2011. [doi:10.1561/1900000028](https://doi.org/10.1561/1900000028)
[^3]: Evan Jones. [Why databases use ordered indexes but programming uses hash tables](https://www.evanjones.ca/ordered-vs-unordered-indexes.html). *evanjones.ca*, December 2019. Archived at [perma.cc/NJX8-3ZZD](https://perma.cc/NJX8-3ZZD)
[^4]: Branimir Lambov. [CEP-25: Trie-indexed SSTable format](https://cwiki.apache.org/confluence/display/CASSANDRA/CEP-25%3A%2BTrie-indexed%2BSSTable%2Bformat). *cwiki.apache.org*, November 2022. Archived at [perma.cc/HD7W-PW8U](https://perma.cc/HD7W-PW8U). Linked Google Doc archived at [perma.cc/UL6C-AAAE](https://perma.cc/UL6C-AAAE)
[^5]: Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest, and Clifford Stein: *Introduction to Algorithms*, 3rd edition. MIT Press, 2009. ISBN: 978-0-262-53305-8
[^6]: Branimir Lambov. [Trie Memtables in Cassandra](https://www.vldb.org/pvldb/vol15/p3359-lambov.pdf). *Proceedings of the VLDB Endowment*, volume 15, issue 12, pages 3359–3371, August 2022. [doi:10.14778/3554821.3554828](https://doi.org/10.14778/3554821.3554828)
[^7]: Dhruba Borthakur. [The History of RocksDB](https://rocksdb.blogspot.com/2013/11/the-history-of-rocksdb.html). *rocksdb.blogspot.com*, November 2013. Archived at [perma.cc/Z7C5-JPSP](https://perma.cc/Z7C5-JPSP)
[^8]: Matteo Bertozzi. [Apache HBase I/O – HFile](https://blog.cloudera.com/apache-hbase-i-o-hfile/). *blog.cloudera.com*, June 2012. Archived at [perma.cc/U9XH-L2KL](https://perma.cc/U9XH-L2KL)
[^9]: Fay Chang, Jeffrey Dean, Sanjay Ghemawat, Wilson C. Hsieh, Deborah A. Wallach, Mike Burrows, Tushar Chandra, Andrew Fikes, and Robert E. Gruber. [Bigtable: A Distributed Storage System for Structured Data](https://research.google/pubs/pub27898/). At *7th USENIX Symposium on Operating System Design and Implementation* (OSDI), November 2006.
[^10]: Patrick O’Neil, Edward Cheng, Dieter Gawlick, and Elizabeth O’Neil. [The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf). *Acta Informatica*, volume 33, issue 4, pages 351–385, June 1996. [doi:10.1007/s002360050048](https://doi.org/10.1007/s002360050048)
[^11]: Mendel Rosenblum and John K. Ousterhout. [The Design and Implementation of a Log-Structured File System](https://research.cs.wisc.edu/areas/os/Qual/papers/lfs.pdf). *ACM Transactions on Computer Systems*, volume 10, issue 1, pages 26–52, February 1992. [doi:10.1145/146941.146943](https://doi.org/10.1145/146941.146943)
[^12]: Michael Armbrust, Tathagata Das, Liwen Sun, Burak Yavuz, Shixiong Zhu, Mukul Murthy, Joseph Torres, Herman van Hovell, Adrian Ionescu, Alicja Łuszczak, Michał Świtakowski, Michał Szafrański, Xiao Li, Takuya Ueshin, Mostafa Mokhtar, Peter Boncz, Ali Ghodsi, Sameer Paranjpye, Pieter Senster, Reynold Xin, and Matei Zaharia. [Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores](https://vldb.org/pvldb/vol13/p3411-armbrust.pdf). *Proceedings of the VLDB Endowment*, volume 13, issue 12, pages 3411–3424, August 2020. [doi:10.14778/3415478.3415560](https://doi.org/10.14778/3415478.3415560)
[^13]: Burton H. Bloom. [Space/Time Trade-offs in Hash Coding with Allowable Errors](https://people.cs.umass.edu/~emery/classes/cmpsci691st/readings/Misc/p422-bloom.pdf). *Communications of the ACM*, volume 13, issue 7, pages 422–426, July 1970. [doi:10.1145/362686.362692](https://doi.org/10.1145/362686.362692)
[^14]: Adam Kirsch and Michael Mitzenmacher. [Less Hashing, Same Performance: Building a Better Bloom Filter](https://www.eecs.harvard.edu/~michaelm/postscripts/tr-02-05.pdf). *Random Structures & Algorithms*, volume 33, issue 2, pages 187–218, September 2008. [doi:10.1002/rsa.20208](https://doi.org/10.1002/rsa.20208)
[^15]: Thomas Hurst. [Bloom Filter Calculator](https://hur.st/bloomfilter/). *hur.st*, September 2023. Archived at [perma.cc/L3AV-6VC2](https://perma.cc/L3AV-6VC2)
[^16]: Chen Luo and Michael J. Carey. [LSM-based storage techniques: a survey](https://arxiv.org/abs/1812.07527). *The VLDB Journal*, volume 29, pages 393–418, July 2019. [doi:10.1007/s00778-019-00555-y](https://doi.org/10.1007/s00778-019-00555-y)
[^17]: Subhadeep Sarkar and Manos Athanassoulis. [Dissecting, Designing, and Optimizing LSM-based Data Stores](https://www.youtube.com/watch?v=hkMkBZn2mGs). Tutorial at *ACM International Conference on Management of Data* (SIGMOD), June 2022. Slides archived at [perma.cc/93B3-E827](https://perma.cc/93B3-E827)
[^18]: Mark Callaghan. [Name that compaction algorithm](https://smalldatum.blogspot.com/2018/08/name-that-compaction-algorithm.html). *smalldatum.blogspot.com*, August 2018. Archived at [perma.cc/CN4M-82DY](https://perma.cc/CN4M-82DY)
[^19]: Prashanth Rao. [Embedded databases (1): The harmony of DuckDB, KùzuDB and LanceDB](https://thedataquarry.com/posts/embedded-db-1/). *thedataquarry.com*, August 2023. Archived at [perma.cc/PA28-2R35](https://perma.cc/PA28-2R35)
[^20]: Hacker News discussion. [Bluesky migrates to single-tenant SQLite](https://news.ycombinator.com/item?id=38171322). *news.ycombinator.com*, October 2023. Archived at [perma.cc/69LM-5P6X](https://perma.cc/69LM-5P6X)
[^21]: Rudolf Bayer and Edward M. McCreight. [Organization and Maintenance of Large Ordered Indices](https://dl.acm.org/doi/pdf/10.1145/1734663.1734671). Boeing Scientific Research Laboratories, Mathematical and Information Sciences Laboratory, report no. 20, July 1970. [doi:10.1145/1734663.1734671](https://doi.org/10.1145/1734663.1734671)
[^22]: Douglas Comer. [The Ubiquitous B-Tree](https://web.archive.org/web/20170809145513id_/http%3A//sites.fas.harvard.edu/~cs165/papers/comer.pdf). *ACM Computing Surveys*, volume 11, issue 2, pages 121–137, June 1979. [doi:10.1145/356770.356776](https://doi.org/10.1145/356770.356776)
[^23]: Alex Miller. [Torn Write Detection and Protection](https://transactional.blog/blog/2025-torn-writes). *transactional.blog*, April 2025. Archived at [perma.cc/G7EB-33EW](https://perma.cc/G7EB-33EW)
[^24]: C. Mohan and Frank Levine. [ARIES/IM: An Efficient and High Concurrency Index Management Method Using Write-Ahead Logging](https://ics.uci.edu/~cs223/papers/p371-mohan.pdf). At *ACM International Conference on Management of Data* (SIGMOD), June 1992. [doi:10.1145/130283.130338](https://doi.org/10.1145/130283.130338)
[^25]: Hironobu Suzuki. [The Internals of PostgreSQL](https://www.interdb.jp/pg/). *interdb.jp*, 2017.
[^26]: Howard Chu. [LDAP at Lightning Speed](https://buildstuff14.sched.com/event/08a1a368e272eb599a52e08b4c3c779d). At *Build Stuff ’14*, November 2014. Archived at [perma.cc/GB6Z-P8YH](https://perma.cc/GB6Z-P8YH)
[^27]: Manos Athanassoulis, Michael S. Kester, Lukas M. Maas, Radu Stoica, Stratos Idreos, Anastasia Ailamaki, and Mark Callaghan. [Designing Access Methods: The RUM Conjecture](https://openproceedings.org/2016/conf/edbt/paper-12.pdf). At *19th International Conference on Extending Database Technology* (EDBT), March 2016. [doi:10.5441/002/edbt.2016.42](https://doi.org/10.5441/002/edbt.2016.42)
[^28]: Ben Stopford. [Log Structured Merge Trees](http://www.benstopford.com/2015/02/14/log-structured-merge-trees/). *benstopford.com*, February 2015. Archived at [perma.cc/E5BV-KUJ6](https://perma.cc/E5BV-KUJ6)
[^29]: Mark Callaghan. [The Advantages of an LSM vs a B-Tree](https://smalldatum.blogspot.com/2016/01/summary-of-advantages-of-lsm-vs-b-tree.html). *smalldatum.blogspot.co.uk*, January 2016. Archived at [perma.cc/3TYZ-EFUD](https://perma.cc/3TYZ-EFUD)
[^30]: Oana Balmau, Florin Dinu, Willy Zwaenepoel, Karan Gupta, Ravishankar Chandhiramoorthi, and Diego Didona. [SILK: Preventing Latency Spikes in Log-Structured Merge Key-Value Stores](https://www.usenix.org/conference/atc19/presentation/balmau). At *USENIX Annual Technical Conference*, July 2019.
[^31]: Igor Canadi, Siying Dong, Mark Callaghan, et al. [RocksDB Tuning Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide). *github.com*, 2023. Archived at [perma.cc/UNY4-MK6C](https://perma.cc/UNY4-MK6C)
[^32]: Gabriel Haas and Viktor Leis. [What Modern NVMe Storage Can Do, and How to Exploit it: High-Performance I/O for High-Performance Storage Engines](https://www.vldb.org/pvldb/vol16/p2090-haas.pdf). *Proceedings of the VLDB Endowment*, volume 16, issue 9, pages 2090-2102. [doi:10.14778/3598581.3598584](https://doi.org/10.14778/3598581.3598584)
[^33]: Emmanuel Goossaert. [Coding for SSDs](https://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/). *codecapsule.com*, February 2014.
[^34]: Jack Vanlightly. [Is sequential IO dead in the era of the NVMe drive?](https://jack-vanlightly.com/blog/2023/5/9/is-sequential-io-dead-in-the-era-of-the-nvme-drive) *jack-vanlightly.com*, May 2023. Archived at [perma.cc/7TMZ-TAPU](https://perma.cc/7TMZ-TAPU)
[^35]: Alibaba Cloud Storage Team. [Storage System Design Analysis: Factors Affecting NVMe SSD Performance (2)](https://www.alibabacloud.com/blog/594376). *alibabacloud.com*, January 2019. Archived at [archive.org](https://web.archive.org/web/20230510065132/https%3A//www.alibabacloud.com/blog/594376)
[^36]: Xiao-Yu Hu and Robert Haas. [The Fundamental Limit of Flash Random Write Performance: Understanding, Analysis and Performance Modelling](https://dominoweb.draco.res.ibm.com/reports/rz3771.pdf). *dominoweb.draco.res.ibm.com*, March 2010. Archived at [perma.cc/8JUL-4ZDS](https://perma.cc/8JUL-4ZDS)
[^37]: Lanyue Lu, Thanumalayan Sankaranarayana Pillai, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. [WiscKey: Separating Keys from Values in SSD-conscious Storage](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf). At *4th USENIX Conference on File and Storage Technologies* (FAST), February 2016.
[^38]: Peter Zaitsev. [Innodb Double Write](https://www.percona.com/blog/innodb-double-write/). *percona.com*, August 2006. Archived at [perma.cc/NT4S-DK7T](https://perma.cc/NT4S-DK7T)
[^39]: Tomas Vondra. [On the Impact of Full-Page Writes](https://www.2ndquadrant.com/en/blog/on-the-impact-of-full-page-writes/). *2ndquadrant.com*, November 2016. Archived at [perma.cc/7N6B-CVL3](https://perma.cc/7N6B-CVL3)
[^40]: Mark Callaghan. [Read, write & space amplification - B-Tree vs LSM](https://smalldatum.blogspot.com/2015/11/read-write-space-amplification-b-tree.html). *smalldatum.blogspot.com*, November 2015. Archived at [perma.cc/S487-WK5P](https://perma.cc/S487-WK5P)
[^41]: Mark Callaghan. [Choosing Between Efficiency and Performance with RocksDB](https://www.youtube.com/watch?v=tgzkgZVXKB4). At *Code Mesh*, November 2016. Video at [youtube.com/watch?v=tgzkgZVXKB4](https://www.youtube.com/watch?v=tgzkgZVXKB4)
[^42]: Subhadeep Sarkar, Tarikul Islam Papon, Dimitris Staratzis, Zichen Zhu, and Manos Athanassoulis. [Enabling Timely and Persistent Deletion in LSM-Engines](https://subhadeep.net/assets/fulltext/Enabling_Timely_and_Persistent_Deletion_in_LSM-Engines.pdf). *ACM Transactions on Database Systems*, volume 48, issue 3, article no. 8, August 2023. [doi:10.1145/3599724](https://doi.org/10.1145/3599724)
[^43]: Lukas Fittl. [Postgres vs. SQL Server: B-Tree Index Differences & the Benefit of Deduplication](https://pganalyze.com/blog/postgresql-vs-sql-server-btree-index-deduplication). *pganalyze.com*, April 2025. Archived at [perma.cc/XY6T-LTPX](https://perma.cc/XY6T-LTPX)
[^44]: Drew Silcock. [How Postgres stores data on disk – this one’s a page turner](https://drew.silcock.dev/blog/how-postgres-stores-data-on-disk/). *drew.silcock.dev*, August 2024. Archived at [perma.cc/8K7K-7VJ2](https://perma.cc/8K7K-7VJ2)
[^45]: Joe Webb. [Using Covering Indexes to Improve Query Performance](https://www.red-gate.com/simple-talk/databases/sql-server/learn/using-covering-indexes-to-improve-query-performance/). *simple-talk.com*, September 2008. Archived at [perma.cc/6MEZ-R5VR](https://perma.cc/6MEZ-R5VR)
[^46]: Michael Stonebraker, Samuel Madden, Daniel J. Abadi, Stavros Harizopoulos, Nabil Hachem, and Pat Helland. [The End of an Architectural Era (It’s Time for a Complete Rewrite)](https://vldb.org/conf/2007/papers/industrial/p1150-stonebraker.pdf). At *33rd International Conference on Very Large Data Bases* (VLDB), September 2007.
[^47]: [VoltDB Technical Overview White Paper](https://www.voltactivedata.com/wp-content/uploads/2017/03/hv-white-paper-voltdb-technical-overview.pdf). VoltDB, 2017. Archived at [perma.cc/B9SF-SK5G](https://perma.cc/B9SF-SK5G)
[^48]: Stephen M. Rumble, Ankita Kejriwal, and John K. Ousterhout. [Log-Structured Memory for DRAM-Based Storage](https://www.usenix.org/system/files/conference/fast14/fast14-paper_rumble.pdf). At *12th USENIX Conference on File and Storage Technologies* (FAST), February 2014.
[^49]: Stavros Harizopoulos, Daniel J. Abadi, Samuel Madden, and Michael Stonebraker. [OLTP Through the Looking Glass, and What We Found There](https://hstore.cs.brown.edu/papers/hstore-lookingglass.pdf). At *ACM International Conference on Management of Data* (SIGMOD), June 2008. [doi:10.1145/1376616.1376713](https://doi.org/10.1145/1376616.1376713)
[^50]: Per-Åke Larson, Cipri Clinciu, Campbell Fraser, Eric N. Hanson, Mostafa Mokhtar, Michal Nowakiewicz, Vassilis Papadimos, Susan L. Price, Srikumar Rangarajan, Remus Rusanu, and Mayukh Saubhasik. [Enhancements to SQL Server Column Stores](https://web.archive.org/web/20131203001153id_/http%3A//research.microsoft.com/pubs/193599/Apollo3%20-%20Sigmod%202013%20-%20final.pdf). At *ACM International Conference on Management of Data* (SIGMOD), June 2013. [doi:10.1145/2463676.2463708](https://doi.org/10.1145/2463676.2463708)
[^51]: Franz Färber, Norman May, Wolfgang Lehner, Philipp Große, Ingo Müller, Hannes Rauhe, and Jonathan Dees. [The SAP HANA Database – An Architecture Overview](https://web.archive.org/web/20220208081111id_/http%3A//sites.computer.org/debull/A12mar/hana.pdf). *IEEE Data Engineering Bulletin*, volume 35, issue 1, pages 28–33, March 2012.
[^52]: Michael Stonebraker. [The Traditional RDBMS Wisdom Is (Almost Certainly) All Wrong](https://slideshot.epfl.ch/talks/166). Presentation at *EPFL*, May 2013.
[^53]: Adam Prout, Szu-Po Wang, Joseph Victor, Zhou Sun, Yongzhu Li, Jack Chen, Evan Bergeron, Eric Hanson, Robert Walzer, Rodrigo Gomes, and Nikita Shamgunov. [Cloud-Native Transactions and Analytics in SingleStore](https://dl.acm.org/doi/pdf/10.1145/3514221.3526055). At *ACM International Conference on Management of Data* (SIGMOD), June 2022. [doi:10.1145/3514221.3526055](https://doi.org/10.1145/3514221.3526055)
[^54]: Tino Tereshko and Jordan Tigani. [BigQuery under the hood](https://cloud.google.com/blog/products/bigquery/bigquery-under-the-hood). *cloud.google.com*, January 2016. Archived at [perma.cc/WP2Y-FUCF](https://perma.cc/WP2Y-FUCF)
[^55]: Wes McKinney. [The Road to Composable Data Systems: Thoughts on the Last 15 Years and the Future](https://wesmckinney.com/blog/looking-back-15-years/). *wesmckinney.com*, September 2023. Archived at [perma.cc/6L2M-GTJX](https://perma.cc/6L2M-GTJX)
[^56]: Michael Stonebraker, Daniel J. Abadi, Adam Batkin, Xuedong Chen, Mitch Cherniack, Miguel Ferreira, Edmond Lau, Amerson Lin, Sam Madden, Elizabeth O’Neil, Pat O’Neil, Alex Rasin, Nga Tran, and Stan Zdonik. [C-Store: A Column-oriented DBMS](https://www.vldb.org/archives/website/2005/program/paper/thu/p553-stonebraker.pdf). At *31st International Conference on Very Large Data Bases* (VLDB), pages 553–564, September 2005.
[^57]: Julien Le Dem. [Dremel Made Simple with Parquet](https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet.html). *blog.twitter.com*, September 2013.
[^58]: Sergey Melnik, Andrey Gubarev, Jing Jing Long, Geoffrey Romer, Shiva Shivakumar, Matt Tolton, and Theo Vassilakis. [Dremel: Interactive Analysis of Web-Scale Datasets](https://vldb.org/pvldb/vol3/R29.pdf). At *36th International Conference on Very Large Data Bases* (VLDB), pages 330–339, September 2010. [doi:10.14778/1920841.1920886](https://doi.org/10.14778/1920841.1920886)
[^59]: Joe Kearney. [Understanding Record Shredding: storing nested data in columns](https://www.joekearney.co.uk/posts/understanding-record-shredding). *joekearney.co.uk*, December 2016. Archived at [perma.cc/ZD5N-AX5D](https://perma.cc/ZD5N-AX5D)
[^60]: Jamie Brandon. [A shallow survey of OLAP and HTAP query engines](https://www.scattered-thoughts.net/writing/a-shallow-survey-of-olap-and-htap-query-engines). *scattered-thoughts.net*, September 2023. Archived at [perma.cc/L3KH-J4JF](https://perma.cc/L3KH-J4JF)
[^61]: Benoit Dageville, Thierry Cruanes, Marcin Zukowski, Vadim Antonov, Artin Avanes, Jon Bock, Jonathan Claybaugh, Daniel Engovatov, Martin Hentschel, Jiansheng Huang, Allison W. Lee, Ashish Motivala, Abdul Q. Munir, Steven Pelley, Peter Povinec, Greg Rahn, Spyridon Triantafyllis, and Philipp Unterbrunner. [The Snowflake Elastic Data Warehouse](https://dl.acm.org/doi/pdf/10.1145/2882903.2903741). At *ACM International Conference on Management of Data* (SIGMOD), pages 215–226, June 2016. [doi:10.1145/2882903.2903741](https://doi.org/10.1145/2882903.2903741)
[^62]: Mark Raasveldt and Hannes Mühleisen. [Data Management for Data Science Towards Embedded Analytics](https://duckdb.org/pdf/CIDR2020-raasveldt-muehleisen-duckdb.pdf). At *10th Conference on Innovative Data Systems Research* (CIDR), January 2020.
[^63]: Jean-François Im, Kishore Gopalakrishna, Subbu Subramaniam, Mayank Shrivastava, Adwait Tumbde, Xiaotian Jiang, Jennifer Dai, Seunghyun Lee, Neha Pawar, Jialiang Li, and Ravi Aringunram. [Pinot: Realtime OLAP for 530 Million Users](https://cwiki.apache.org/confluence/download/attachments/103092375/Pinot.pdf). At *ACM International Conference on Management of Data* (SIGMOD), pages 583–594, May 2018. [doi:10.1145/3183713.3190661](https://doi.org/10.1145/3183713.3190661)
[^64]: Fangjin Yang, Eric Tschetter, Xavier Léauté, Nelson Ray, Gian Merlino, and Deep Ganguli. [Druid: A Real-time Analytical Data Store](https://static.druid.io/docs/druid.pdf). At *ACM International Conference on Management of Data* (SIGMOD), June 2014. [doi:10.1145/2588555.2595631](https://doi.org/10.1145/2588555.2595631)
[^65]: Chunwei Liu, Anna Pavlenko, Matteo Interlandi, and Brandon Haynes. [Deep Dive into Common Open Formats for Analytical DBMSs](https://www.vldb.org/pvldb/vol16/p3044-liu.pdf). *Proceedings of the VLDB Endowment*, volume 16, issue 11, pages 3044–3056, July 2023. [doi:10.14778/3611479.3611507](https://doi.org/10.14778/3611479.3611507)
[^66]: Xinyu Zeng, Yulong Hui, Jiahong Shen, Andrew Pavlo, Wes McKinney, and Huanchen Zhang. [An Empirical Evaluation of Columnar Storage Formats](https://www.vldb.org/pvldb/vol17/p148-zeng.pdf). *Proceedings of the VLDB Endowment*, volume 17, issue 2, pages 148–161. [doi:10.14778/3626292.3626298](https://doi.org/10.14778/3626292.3626298)
[^67]: Weston Pace. [Lance v2: A columnar container format for modern data](https://blog.lancedb.com/lance-v2/). *blog.lancedb.com*, April 2024. Archived at [perma.cc/ZK3Q-S9VJ](https://perma.cc/ZK3Q-S9VJ)
[^68]: Yoav Helfman. [Nimble, A New Columnar File Format](https://www.youtube.com/watch?v=bISBNVtXZ6M). At *VeloxCon*, April 2024.
[^69]: Wes McKinney. [Apache Arrow: High-Performance Columnar Data Framework](https://www.youtube.com/watch?v=YhF8YR0OEFk). At *CMU Database Group – Vaccination Database Tech Talks*, December 2021.
[^70]: Wes McKinney. [Python for Data Analysis, 3rd Edition](https://learning.oreilly.com/library/view/python-for-data/9781098104023/). O’Reilly Media, August 2022. ISBN: 9781098104023
[^71]: Paul Dix. [The Design of InfluxDB IOx: An In-Memory Columnar Database Written in Rust with Apache Arrow](https://www.youtube.com/watch?v=_zbwz-4RDXg). At *CMU Database Group – Vaccination Database Tech Talks*, May 2021.
[^72]: Carlota Soto and Mike Freedman. [Building Columnar Compression for Large PostgreSQL Databases](https://www.timescale.com/blog/building-columnar-compression-in-a-row-oriented-database/). *timescale.com*, March 2024. Archived at [perma.cc/7KTF-V3EH](https://perma.cc/7KTF-V3EH)
[^73]: Daniel Lemire, Gregory Ssi‐Yan‐Kai, and Owen Kaser. [Consistently faster and smaller compressed bitmaps with Roaring](https://arxiv.org/pdf/1603.06549). *Software: Practice and Experience*, volume 46, issue 11, pages 1547–1569, November 2016. [doi:10.1002/spe.2402](https://doi.org/10.1002/spe.2402)
[^74]: Jaz Volpert. [An entire Social Network in 1.6GB (GraphD Part 2)](https://jazco.dev/2024/04/20/roaring-bitmaps/). *jazco.dev*, April 2024. Archived at [perma.cc/L27Z-QVMG](https://perma.cc/L27Z-QVMG)
[^75]: Daniel J. Abadi, Peter Boncz, Stavros Harizopoulos, Stratos Idreos, and Samuel Madden. [The Design and Implementation of Modern Column-Oriented Database Systems](https://www.cs.umd.edu/~abadi/papers/abadi-column-stores.pdf). *Foundations and Trends in Databases*, volume 5, issue 3, pages 197–280, December 2013. [doi:10.1561/1900000024](https://doi.org/10.1561/1900000024)
[^76]: Andrew Lamb, Matt Fuller, Ramakrishna Varadarajan, Nga Tran, Ben Vandiver, Lyric Doshi, and Chuck Bear. [The Vertica Analytic Database: C-Store 7 Years Later](https://vldb.org/pvldb/vol5/p1790_andrewlamb_vldb2012.pdf). *Proceedings of the VLDB Endowment*, volume 5, issue 12, pages 1790–1801, August 2012. [doi:10.14778/2367502.2367518](https://doi.org/10.14778/2367502.2367518)
[^77]: Timo Kersten, Viktor Leis, Alfons Kemper, Thomas Neumann, Andrew Pavlo, and Peter Boncz. [Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask](https://www.vldb.org/pvldb/vol11/p2209-kersten.pdf). *Proceedings of the VLDB Endowment*, volume 11, issue 13, pages 2209–2222, September 2018. [doi:10.14778/3275366.3284966](https://doi.org/10.14778/3275366.3284966)
[^78]: Forrest Smith. [Memory Bandwidth Napkin Math](https://www.forrestthewoods.com/blog/memory-bandwidth-napkin-math/). *forrestthewoods.com*, February 2020. Archived at [perma.cc/Y8U4-PS7N](https://perma.cc/Y8U4-PS7N)
[^79]: Peter Boncz, Marcin Zukowski, and Niels Nes. [MonetDB/X100: Hyper-Pipelining Query Execution](https://www.cidrdb.org/cidr2005/papers/P19.pdf). At *2nd Biennial Conference on Innovative Data Systems Research* (CIDR), January 2005.
[^80]: Jingren Zhou and Kenneth A. Ross. [Implementing Database Operations Using SIMD Instructions](https://www1.cs.columbia.edu/~kar/pubsk/simd.pdf). At *ACM International Conference on Management of Data* (SIGMOD), pages 145–156, June 2002. [doi:10.1145/564691.564709](https://doi.org/10.1145/564691.564709)
[^81]: Kevin Bartley. [OLTP Queries: Transfer Expensive Workloads to Materialize](https://materialize.com/blog/oltp-queries/). *materialize.com*, August 2024. Archived at [perma.cc/4TYM-TYD8](https://perma.cc/4TYM-TYD8)
[^82]: Jim Gray, Surajit Chaudhuri, Adam Bosworth, Andrew Layman, Don Reichart, Murali Venkatrao, Frank Pellow, and Hamid Pirahesh. [Data Cube: A Relational Aggregation Operator Generalizing Group-By, Cross-Tab, and Sub-Totals](https://arxiv.org/pdf/cs/0701155). *Data Mining and Knowledge Discovery*, volume 1, issue 1, pages 29–53, March 2007. [doi:10.1023/A:1009726021843](https://doi.org/10.1023/A%3A1009726021843)
[^83]: Frank Ramsak, Volker Markl, Robert Fenk, Martin Zirkel, Klaus Elhardt, and Rudolf Bayer. [Integrating the UB-Tree into a Database System Kernel](https://www.vldb.org/conf/2000/P263.pdf). At *26th International Conference on Very Large Data Bases* (VLDB), September 2000.
[^84]: Octavian Procopiuc, Pankaj K. Agarwal, Lars Arge, and Jeffrey Scott Vitter. [Bkd-Tree: A Dynamic Scalable kd-Tree](https://users.cs.duke.edu/~pankaj/publications/papers/bkd-sstd.pdf). At *8th International Symposium on Spatial and Temporal Databases* (SSTD), pages 46–65, July 2003. [doi:10.1007/978-3-540-45072-6\_4](https://doi.org/10.1007/978-3-540-45072-6_4)
[^85]: Joseph M. Hellerstein, Jeffrey F. Naughton, and Avi Pfeffer. [Generalized Search Trees for Database Systems](https://dsf.berkeley.edu/papers/vldb95-gist.pdf). At *21st International Conference on Very Large Data Bases* (VLDB), September 1995.
[^86]: Isaac Brodsky. [H3: Uber’s Hexagonal Hierarchical Spatial Index](https://www.uber.com/blog/h3/). *eng.uber.com*, June 2018. Archived at [archive.org](https://web.archive.org/web/20240722003854/https%3A//www.uber.com/blog/h3/)
[^87]: Robert Escriva, Bernard Wong, and Emin Gün Sirer. [HyperDex: A Distributed, Searchable Key-Value Store](https://www.cs.princeton.edu/courses/archive/fall13/cos518/papers/hyperdex.pdf). At *ACM SIGCOMM Conference*, August 2012. [doi:10.1145/2377677.2377681](https://doi.org/10.1145/2377677.2377681)
[^88]: Christopher D. Manning, Prabhakar Raghavan, and Hinrich Schütze. [*Introduction to Information Retrieval*](https://nlp.stanford.edu/IR-book/). Cambridge University Press, 2008. ISBN: 978-0-521-86571-5, available online at [nlp.stanford.edu/IR-book](https://nlp.stanford.edu/IR-book/)
[^89]: Jianguo Wang, Chunbin Lin, Yannis Papakonstantinou, and Steven Swanson. [An Experimental Study of Bitmap Compression vs. Inverted List Compression](https://cseweb.ucsd.edu/~swanson/papers/SIGMOD2017-ListCompression.pdf). At *ACM International Conference on Management of Data* (SIGMOD), pages 993–1008, May 2017. [doi:10.1145/3035918.3064007](https://doi.org/10.1145/3035918.3064007)
[^90]: Adrien Grand. [What is in a Lucene Index?](https://speakerdeck.com/elasticsearch/what-is-in-a-lucene-index) At *Lucene/Solr Revolution*, November 2013. Archived at [perma.cc/Z7QN-GBYY](https://perma.cc/Z7QN-GBYY)
[^91]: Michael McCandless. [Visualizing Lucene’s Segment Merges](https://blog.mikemccandless.com/2011/02/visualizing-lucenes-segment-merges.html). *blog.mikemccandless.com*, February 2011. Archived at [perma.cc/3ZV8-72W6](https://perma.cc/3ZV8-72W6)
[^92]: Lukas Fittl. [Understanding Postgres GIN Indexes: The Good and the Bad](https://pganalyze.com/blog/gin-index). *pganalyze.com*, December 2021. Archived at [perma.cc/V3MW-26H6](https://perma.cc/V3MW-26H6)
[^93]: Jimmy Angelakos. [The State of (Full) Text Search in PostgreSQL 12](https://www.youtube.com/watch?v=c8IrUHV70KQ). At *FOSDEM*, February 2020. Archived at [perma.cc/J6US-3WZS](https://perma.cc/J6US-3WZS)
[^94]: Alexander Korotkov. [Index support for regular expression search](https://wiki.postgresql.org/images/6/6c/Index_support_for_regular_expression_search.pdf). At *PGConf.EU Prague*, October 2012. Archived at [perma.cc/5RFZ-ZKDQ](https://perma.cc/5RFZ-ZKDQ)
[^95]: Michael McCandless. [Lucene’s FuzzyQuery Is 100 Times Faster in 4.0](https://blog.mikemccandless.com/2011/03/lucenes-fuzzyquery-is-100-times-faster.html). *blog.mikemccandless.com*, March 2011. Archived at [perma.cc/E2WC-GHTW](https://perma.cc/E2WC-GHTW)
[^96]: Steffen Heinz, Justin Zobel, and Hugh E. Williams. [Burst Tries: A Fast, Efficient Data Structure for String Keys](https://web.archive.org/web/20130903070248id_/http%3A//ww2.cs.mu.oz.au%3A80/~jz/fulltext/acmtois02.pdf). *ACM Transactions on Information Systems*, volume 20, issue 2, pages 192–223, April 2002. [doi:10.1145/506309.506312](https://doi.org/10.1145/506309.506312)
[^97]: Klaus U. Schulz and Stoyan Mihov. [Fast String Correction with Levenshtein Automata](https://dmice.ohsu.edu/bedricks/courses/cs655/pdf/readings/2002_Schulz.pdf). *International Journal on Document Analysis and Recognition*, volume 5, issue 1, pages 67–85, November 2002. [doi:10.1007/s10032-002-0082-8](https://doi.org/10.1007/s10032-002-0082-8)
[^98]: Tomas Mikolov, Kai Chen, Greg Corrado, and Jeffrey Dean. [Efficient Estimation of Word Representations in Vector Space](https://arxiv.org/pdf/1301.3781). At *International Conference on Learning Representations* (ICLR), May 2013. [doi:10.48550/arXiv.1301.3781](https://doi.org/10.48550/arXiv.1301.3781)
[^99]: Jacob Devlin, Ming-Wei Chang, Kenton Lee, and Kristina Toutanova. [BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding](https://arxiv.org/pdf/1810.04805). At *Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies*, volume 1, pages 4171–4186, June 2019. [doi:10.18653/v1/N19-1423](https://doi.org/10.18653/v1/N19-1423)
[^100]: Alec Radford, Karthik Narasimhan, Tim Salimans, and Ilya Sutskever. [Improving Language Understanding by Generative Pre-Training](https://cdn.openai.com/research-covers/language-unsupervised/language_understanding_paper.pdf). *openai.com*, June 2018. Archived at [perma.cc/5N3C-DJ4C](https://perma.cc/5N3C-DJ4C)
[^101]: Matthijs Douze, Maria Lomeli, and Lucas Hosseini. [Faiss indexes](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes). *github.com*, August 2024. Archived at [perma.cc/2EWG-FPBS](https://perma.cc/2EWG-FPBS)
[^102]: Varik Matevosyan. [Understanding pgvector’s HNSW Index Storage in Postgres](https://lantern.dev/blog/pgvector-storage). *lantern.dev*, August 2024. Archived at [perma.cc/B2YB-JB59](https://perma.cc/B2YB-JB59)
[^103]: Dmitry Baranchuk, Artem Babenko, and Yury Malkov. [Revisiting the Inverted Indices for Billion-Scale Approximate Nearest Neighbors](https://arxiv.org/pdf/1802.02422). At *European Conference on Computer Vision* (ECCV), pages 202–216, September 2018. [doi:10.1007/978-3-030-01258-8\_13](https://doi.org/10.1007/978-3-030-01258-8_13)
[^104]: Yury A. Malkov and Dmitry A. Yashunin. [Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs](https://arxiv.org/pdf/1603.09320). *IEEE Transactions on Pattern Analysis and Machine Intelligence*, volume 42, issue 4, pages 824–836, April 2020. [doi:10.1109/TPAMI.2018.2889473](https://doi.org/10.1109/TPAMI.2018.2889473)
