---
translationKey: ddia-v2-ch3
title: Mô hình dữ liệu và ngôn ngữ truy vấn
book_kind: chapter
book_number: "3"
book_part: I
weight: 103
breadcrumbs: false
---

<a id="ch_datamodels"></a>

![](/map/ch02.png)

> *Giới hạn của ngôn ngữ của tôi nghĩa là giới hạn của thế giới của tôi.*
>
> Ludwig Wittgenstein, *Tractatus Logico-Philosophicus* (1922)

Mô hình dữ liệu có lẽ là phần quan trọng nhất của phát triển phần mềm, vì chúng có ảnh hưởng sâu rộng: không chỉ đến cách phần mềm được viết, mà còn đến cách chúng ta *suy nghĩ về vấn đề* cần giải quyết.

Hầu hết ứng dụng được xây dựng bằng cách xếp chồng các lớp mô hình dữ liệu. Câu hỏi then chốt với mỗi lớp là: nó được *biểu diễn* bằng mô hình dữ liệu của lớp thấp hơn như thế nào? Chẳng hạn:

1. Là lập trình viên ứng dụng, bạn quan sát thế giới thực (với con người, tổ chức, hàng hóa, hành động, dòng tiền, cảm biến, v.v.) và mô hình hóa nó bằng các đối tượng hoặc cấu trúc dữ liệu, cùng các API để thao tác trên những cấu trúc đó. Các cấu trúc này thường đặc thù cho từng ứng dụng.
2. Khi cần lưu trữ các cấu trúc dữ liệu đó, bạn biểu diễn chúng bằng một mô hình dữ liệu đa dụng, như tài liệu JSON hoặc XML, các bảng trong cơ sở dữ liệu quan hệ, hay các đỉnh và cạnh trong một đồ thị. Những mô hình dữ liệu này chính là chủ đề của chương này.
3. Các kỹ sư xây dựng phần mềm cơ sở dữ liệu quyết định cách biểu diễn dữ liệu tài liệu, quan hệ hay đồ thị dưới dạng các byte trong bộ nhớ, trên đĩa hoặc trên mạng. Cách biểu diễn này giúp dữ liệu có thể được truy vấn, tìm kiếm, thao tác và xử lý theo nhiều cách. Ta sẽ bàn về thiết kế của các storage engine này ở [Chương 4](/ch4#ch_storage).
4. Ở tầng thấp hơn nữa, các kỹ sư phần cứng đã nghĩ ra cách biểu diễn byte bằng dòng điện, xung ánh sáng, từ trường và nhiều thứ khác.

Một ứng dụng phức tạp có thể có nhiều tầng trung gian hơn, chẳng hạn các API xây trên các API khác, nhưng ý tưởng cơ bản vẫn như nhau: mỗi tầng che giấu sự phức tạp của các tầng thấp hơn bằng cách cung cấp một mô hình dữ liệu rõ ràng. Những lớp trừu tượng này cho phép các nhóm người khác nhau, ví dụ kỹ sư của hãng cơ sở dữ liệu và lập trình viên ứng dụng dùng cơ sở dữ liệu đó, cộng tác hiệu quả.

Trong thực tế có nhiều mô hình dữ liệu khác nhau được dùng rộng rãi, thường mỗi loại phục vụ một mục đích. Một số kiểu dữ liệu và truy vấn dễ diễn đạt trong mô hình này nhưng lại vụng về trong mô hình khác. Chương này sẽ so sánh *mô hình quan hệ* (*relational model*), *mô hình tài liệu* (*document model*), *mô hình dữ liệu đồ thị* (*graph data model*), *event sourcing* (*event sourcing*) và *dataframe* (*dataframe*), qua đó khảo sát các đánh đổi. Ta cũng sẽ lướt qua các ngôn ngữ truy vấn dùng cho những mô hình này, để bạn có thể quyết định nên dùng mô hình nào trong từng trường hợp.


> [!TIP] Thuật ngữ: ngôn ngữ truy vấn khai báo
>
> Nhiều ngôn ngữ truy vấn trong chương này (như SQL, Cypher, SPARQL hay Datalog) là ngôn ngữ *khai báo* (*declarative*). Với ngôn ngữ truy vấn khai báo, bạn chỉ cần chỉ rõ mẫu của dữ liệu mong muốn — kết quả phải thỏa mãn điều kiện nào, và dữ liệu cần được biến đổi ra sao (ví dụ sắp xếp, nhóm và tổng hợp) — chứ không cần nói *làm thế nào* để đạt được điều đó. Bộ tối ưu truy vấn của hệ cơ sở dữ liệu sẽ quyết định dùng chỉ mục (*index*) và thuật toán nối nào, cũng như thứ tự thực thi các phần của truy vấn.
>
> Ngược lại, với hầu hết ngôn ngữ lập trình, bạn phải viết một *thuật toán* (*algorithm*), tức là bảo máy tính thực hiện những thao tác nào theo thứ tự nào. Ngôn ngữ truy vấn khai báo thường ngắn gọn và dễ viết hơn thuật toán tường minh, nhưng quan trọng hơn, nó che giấu chi tiết triển khai của bộ máy truy vấn, nhờ đó hệ cơ sở dữ liệu có thể cải thiện hiệu năng mà không cần thay đổi bất kỳ truy vấn nào [^1].
>
> Chẳng hạn, một cơ sở dữ liệu có thể thực thi truy vấn khai báo song song trên nhiều lõi CPU và nhiều máy mà bạn không phải lo cách hiện thực việc song song đó [^2]. Nếu tự viết thuật toán bằng tay, việc hiện thực thực thi song song sẽ tốn rất nhiều công sức.


## Mô hình quan hệ và mô hình tài liệu {#sec_datamodels_history}

Mô hình dữ liệu nổi tiếng nhất hiện nay có lẽ là mô hình quan hệ mà SQL sử dụng, do Edgar Codd đề xuất năm 1970 [^3], theo đó dữ liệu được tổ chức thành các *quan hệ* (*relation*, trong SQL gọi là *bảng*, *table*), mỗi quan hệ là một tập không có thứ tự gồm các *bộ* (*tuple*, trong SQL gọi là *hàng*, *row*).

Ban đầu mô hình quan hệ chỉ là một đề xuất lý thuyết, và nhiều người nghi ngờ liệu nó có thể được hiện thực hiệu quả hay không. Tuy nhiên, đến giữa thập niên 1980, hệ quản trị cơ sở dữ liệu quan hệ (RDBMS) và SQL đã trở thành công cụ được lựa chọn của hầu hết những người cần lưu trữ và truy vấn dữ liệu có cấu trúc tương đối đều đặn. Nhiều thập kỷ sau, dữ liệu quan hệ vẫn thống trị nhiều tình huống quản lý dữ liệu, chẳng hạn phân tích kinh doanh (xem [“Star schema và snowflake schema: các schema cho phân tích”](/ch3#sec_datamodels_analytics)).

Qua nhiều năm, đã có rất nhiều cách tiếp cận cạnh tranh trong lưu trữ và truy vấn dữ liệu. Từ thập niên 1970 đến đầu thập niên 1980, *mô hình mạng* (*network model*) và *mô hình phân cấp* (*hierarchical model*) là các đối thủ chính của mô hình quan hệ, nhưng cuối cùng đều thất bại. Cơ sở dữ liệu hướng đối tượng xuất hiện rồi biến mất vào cuối thập niên 1980 và đầu thập niên 1990. Cơ sở dữ liệu XML xuất hiện đầu thập niên 2000 nhưng chỉ được dùng trong một số ít tình huống. Mỗi đối thủ của mô hình quan hệ đều từng gây tiếng vang lớn, nhưng không cái nào trụ được lâu [^4]. Thay vào đó, SQL đã tiếp thu thêm các kiểu dữ liệu khác trên nền tảng quan hệ, chẳng hạn bổ sung hỗ trợ XML, JSON và dữ liệu đồ thị [^5].

Đến thập niên 2010, *NoSQL* là từ thời thượng mới nhất nhằm lật đổ vị thế thống trị của cơ sở dữ liệu quan hệ. NoSQL không phải một công nghệ cụ thể mà là một nhóm ý tưởng rộng xoay quanh mô hình dữ liệu mới, tính linh hoạt của schema, khả năng mở rộng và mô hình cấp phép mã nguồn mở. Một số cơ sở dữ liệu khác tự xưng là *NewSQL*, nhằm cung cấp khả năng mở rộng của hệ NoSQL trong khi vẫn giữ mô hình dữ liệu và các đảm bảo giao dịch của cơ sở dữ liệu quan hệ truyền thống. Các ý tưởng NoSQL và NewSQL đã ảnh hưởng sâu sắc đến thiết kế hệ thống dữ liệu, nhưng khi các nguyên tắc ấy được tiếp nhận rộng rãi, bản thân hai thuật ngữ này dần ít được dùng.

Một di sản bền vững của phong trào NoSQL là sự phổ biến của mô hình tài liệu, trong đó dữ liệu thường được biểu diễn bằng JSON. Mô hình này ban đầu được các cơ sở dữ liệu tài liệu chuyên dụng như MongoDB và Couchbase phổ biến, và đến nay hầu hết cơ sở dữ liệu quan hệ cũng đã hỗ trợ JSON. So với schema bảng quan hệ vốn thường bị coi là cứng nhắc và thiếu linh hoạt, tài liệu JSON được xem là linh hoạt hơn.

Dữ liệu tài liệu và dữ liệu quan hệ, bên nào tốt hơn, đã gây ra nhiều tranh luận. Hãy cùng xem xét một số vấn đề then chốt trong đó.

### Sự không khớp đối tượng-quan hệ {#sec_datamodels_document}

Ngày nay phần lớn việc phát triển ứng dụng dùng các ngôn ngữ lập trình hướng đối tượng, dẫn đến một lời chỉ trích phổ biến đối với mô hình dữ liệu SQL: nếu dữ liệu lưu trong các bảng quan hệ, cần có một tầng chuyển đổi vụng về giữa các đối tượng trong mã ứng dụng và mô hình bảng, hàng, cột của cơ sở dữ liệu. Sự rời rạc giữa hai mô hình này đôi khi được gọi là *impedance mismatch*.


> [!NOTE]
> Thuật ngữ *impedance mismatch* được mượn từ điện tử học. Mỗi mạch điện có một trở kháng nhất định (sức cản đối với dòng điện xoay chiều) ở đầu vào và đầu ra. Khi nối đầu ra của mạch này với đầu vào của mạch kia, nếu trở kháng hai bên khớp nhau thì công suất truyền qua chỗ nối sẽ lớn nhất. Trở kháng không khớp có thể gây ra các vấn đề như phản xạ tín hiệu.


#### Ánh xạ đối tượng-quan hệ (ORM) {#object-relational-mapping-orm}

Các framework ánh xạ đối tượng-quan hệ (ORM) như ActiveRecord và Hibernate giảm lượng mã khuôn mẫu cần cho tầng chuyển đổi này, nhưng cũng thường bị chỉ trích [^6]. Một số vấn đề thường gặp:

* Bản thân ORM phức tạp và không thể che giấu hoàn toàn sự khác biệt giữa hai mô hình, nên lập trình viên rốt cuộc vẫn phải nghĩ đồng thời đến cả biểu diễn quan hệ lẫn biểu diễn đối tượng của dữ liệu.
* ORM thường chỉ dùng để phát triển ứng dụng OLTP (xem [“Đặc điểm của xử lý giao dịch và phân tích”](/ch1#sec_introduction_oltp)). Để dữ liệu có thể dùng cho phân tích, kỹ sư dữ liệu vẫn phải làm việc với biểu diễn quan hệ bên dưới, nên dùng ORM không có nghĩa là thiết kế schema quan hệ hết quan trọng.
* Nhiều ORM chỉ hỗ trợ cơ sở dữ liệu OLTP quan hệ. Nếu tổ chức còn dùng các hệ dữ liệu khác như công cụ tìm kiếm, cơ sở dữ liệu đồ thị và hệ NoSQL, sự hỗ trợ của ORM có thể rất thiếu.
* Một số ORM tự động sinh schema quan hệ, nhưng schema sinh ra có thể không thân thiện với người dùng truy cập trực tiếp dữ liệu quan hệ, và có thể kém hiệu quả trên cơ sở dữ liệu bên dưới. Tùy biến cách ORM sinh schema và truy vấn thường khá phức tạp, thậm chí triệt tiêu những lợi ích vốn là lý do dùng ORM.
* ORM dễ khiến ta vô tình viết truy vấn kém hiệu quả, chẳng hạn gây ra *vấn đề truy vấn N+1* (*N+1 query problem*) [^7]. Giả sử bạn muốn hiển thị danh sách bình luận của người dùng trên một trang: bạn dùng một truy vấn lấy *N* bình luận, mỗi bình luận chứa ID tác giả. Để hiển thị tên tác giả, bạn lại phải truy vấn bảng người dùng bằng ID đó. Khi viết SQL bằng tay, nhiều khả năng bạn sẽ nối thẳng bảng người dùng vào truy vấn để mỗi bình luận trả về kèm tên tác giả, còn với ORM, bạn có thể bị truy vấn bảng người dùng riêng cho từng trong *N* bình luận, tổng cộng thực thi *N*+1 truy vấn cơ sở dữ liệu. Cách này chậm hơn nhiều so với thực hiện phép nối trong cơ sở dữ liệu. Để tránh vấn đề này, bạn có thể phải yêu cầu ORM lấy kèm thông tin tác giả cùng lúc với bình luận.

Tuy nhiên, ORM cũng có những ưu điểm riêng:

* Với dữ liệu phù hợp mô hình quan hệ, luôn phải có một bước chuyển đổi nào đó giữa biểu diễn quan hệ được lưu bền vững và biểu diễn đối tượng trong bộ nhớ; ORM giúp giảm lượng mã khuôn mẫu cho việc chuyển đổi này. Các truy vấn phức tạp có thể vẫn cần xử lý bỏ qua ORM, nhưng với các tình huống đơn giản, lặp đi lặp lại thì ORM rất hợp.
* Một số ORM có thể cache kết quả truy vấn cơ sở dữ liệu, giúp giảm tải cho cơ sở dữ liệu.
* ORM cũng có thể hỗ trợ quản lý di chuyển schema và các công việc quản trị cơ sở dữ liệu khác.

#### Mô hình dữ liệu tài liệu cho quan hệ một-nhiều {#the-document-data-model-for-one-to-many-relationships}

Không phải mọi dữ liệu đều phù hợp để biểu diễn dưới dạng quan hệ. Hãy dùng một ví dụ để xem hạn chế của mô hình quan hệ. {{< xref fig="3-1" page="/ch3" anchor="fig_obama_relational" >}}Hình 3-1{{< /xref >}} cho thấy cách biểu diễn một bản sơ yếu lý lịch (hồ sơ LinkedIn) bằng schema quan hệ. Toàn bộ hồ sơ được xác định bởi một định danh duy nhất `user_id`. Các trường như `first_name` và `last_name` chỉ xuất hiện một lần với mỗi người dùng, nên có thể mô hình hóa thành các cột của bảng `users`.

Hầu hết mọi người có nhiều hơn một công việc trong sự nghiệp (nhiều vị trí), số quá trình học vấn của mỗi người cũng khác nhau, và thông tin liên hệ thì có thể có bất kỳ số lượng nào. Các *quan hệ một-nhiều* (*one-to-many relationship*) này có thể được biểu diễn bằng cách đặt vị trí công tác, học vấn và thông tin liên hệ vào các bảng riêng, với khóa ngoại tham chiếu đến bảng `users`, như trong {{< xref fig="3-1" page="/ch3" anchor="fig_obama_relational" >}}Hình 3-1{{< /xref >}}.

{{< fig num="3-1" id="fig_obama_relational" src="/fig/ddia_0301.png" caption="Biểu diễn hồ sơ LinkedIn bằng schema quan hệ." class="ddia-figure ddia-figure--standard" width="1772" height="1414" />}}

Cũng thông tin đó có thể biểu diễn dưới dạng tài liệu JSON, như trong {{< xref eg="3-1" page="/ch3" anchor="fig_obama_json" >}}Ví dụ 3-1{{< /xref >}}. Dạng này có thể tự nhiên hơn và gần với cấu trúc đối tượng trong mã ứng dụng hơn.

{{< eg num="3-1" id="fig_obama_json" caption="Biểu diễn hồ sơ LinkedIn dưới dạng tài liệu JSON" >}}
```json
{
    "user_id": 251,
    "first_name": "Barack",
    "last_name": "Obama",
    "headline": "Former President of the United States of America",
    "region_id": "us:91",
    "photo_url": "/p/7/000/253/05b/308dd6e.jpg",
    "positions": [
        {"job_title": "President", "organization": "United States of America"},
        {"job_title": "US Senator (D-IL)", "organization": "United States Senate"}
    ],
    "education": [
        {"school_name": "Harvard University", "start": 1988, "end": 1991},
        {"school_name": "Columbia University", "start": 1981, "end": 1983}
    ],
    "contact_info": {
        "website": "https://barackobama.com",
        "twitter": "https://twitter.com/barackobama"
    }
}
```
{{< /eg >}}

Một số lập trình viên cho rằng mô hình JSON giảm bớt impedance mismatch giữa mã ứng dụng và tầng lưu trữ. Tuy nhiên, như ta sẽ thấy ở [Chương 5](/ch5#ch_encoding), JSON dùng làm định dạng mã hóa dữ liệu cũng có không ít vấn đề. Việc thiếu schema thường được xem là một ưu điểm của nó. Ta sẽ bàn thêm ở [“Tính linh hoạt của schema trong mô hình tài liệu”](/ch3#sec_datamodels_schema_flexibility).

So với schema nhiều bảng trong {{< xref fig="3-1" page="/ch3" anchor="fig_obama_relational" >}}Hình 3-1{{< /xref >}}, biểu diễn JSON có *tính cục bộ* (*locality*, xem [“Tính cục bộ của dữ liệu khi đọc và ghi”](/ch3#sec_datamodels_document_locality)) tốt hơn. Trong ví dụ quan hệ, để lấy một hồ sơ bạn phải thực hiện nhiều truy vấn (truy vấn từng bảng theo `user_id`), hoặc thực hiện một phép nối nhiều chiều rắc rối giữa bảng `users` và các bảng phụ thuộc của nó [^8]. Với biểu diễn JSON, mọi thông tin liên quan nằm ở một chỗ, nên truy vấn vừa đơn giản vừa nhanh.

Quan hệ một-nhiều giữa hồ sơ với các vị trí công tác, học vấn và thông tin liên hệ tạo thành một cấu trúc cây trong dữ liệu, và biểu diễn JSON làm cấu trúc cây này hiện ra rõ ràng (xem {{< xref fig="3-2" page="/ch3" anchor="fig_json_tree" >}}Hình 3-2{{< /xref >}}).

{{< fig num="3-2" id="fig_json_tree" src="/fig/ddia_0302.png" caption="Quan hệ một-nhiều tạo thành cấu trúc cây." class="ddia-figure ddia-figure--wide" width="1772" height="780" />}}


> [!NOTE]
> Kiểu quan hệ này đôi khi được gọi là *một-vài* (*one-to-few*) thay vì *một-nhiều*, vì một bản sơ yếu lý lịch thường chỉ có vài vị trí công tác [^9] [^10]. Nếu số mục liên quan có thể lớn đến mức đáng kinh ngạc, chẳng hạn bài đăng mạng xã hội của người nổi tiếng có thể nhận hàng nghìn bình luận, thì nhúng tất cả vào cùng một tài liệu sẽ quá cồng kềnh, khi đó nên dùng cách tiếp cận quan hệ như trong {{< xref fig="3-1" page="/ch3" anchor="fig_obama_relational" >}}Hình 3-1{{< /xref >}}.


### Chuẩn hóa, phi chuẩn hóa và phép nối {#sec_datamodels_normalization}

Trong {{< xref eg="3-1" page="/ch3" anchor="fig_obama_json" >}}Ví dụ 3-1{{< /xref >}} ở mục trước, `region_id` được biểu diễn bằng ID thay vì chuỗi văn bản thuần `"Washington, DC, United States"`. Vì sao lại như vậy?

Nếu giao diện người dùng có ô văn bản tự do để nhập khu vực, việc lưu thẳng đầu vào dưới dạng chuỗi văn bản là hợp lý. Tuy nhiên, cung cấp sẵn một danh sách khu vực địa lý được chuẩn hóa, để người dùng chọn qua danh sách thả xuống hoặc tự động hoàn thành, cũng mang lại nhiều lợi ích:

* Định dạng và cách viết nhất quán trên mọi hồ sơ
* Tránh nhập nhằng khi có nhiều nơi trùng tên (nếu chuỗi chỉ là “Washington”, thì là Washington, DC hay bang Washington?)
* Dễ cập nhật: tên chỉ được lưu ở một nơi, nên khi cần đổi (ví dụ đổi tên thành phố do sự kiện chính trị), chỉ cần sửa một chỗ là có hiệu lực trên toàn bộ
* Hỗ trợ bản địa hóa: khi website được dịch sang ngôn ngữ khác, danh sách chuẩn có thể được bản địa hóa để tên khu vực hiển thị bằng ngôn ngữ của người xem
* Cải thiện tìm kiếm: ví dụ, danh sách khu vực có thể ghi nhận việc Washington nằm ở bờ Đông nước Mỹ (điều không thể biết chỉ từ chuỗi `"Washington, DC"`), nhờ vậy khi tìm người ở bờ Đông nước Mỹ, hồ sơ này cũng khớp

Việc lưu ID hay chuỗi văn bản thực chất là quyết định có *chuẩn hóa* (*normalization*) hay không. Khi dùng ID, dữ liệu được chuẩn hóa hơn: thông tin có ý nghĩa với con người (như dòng chữ *Washington, DC*) chỉ lưu một bản, mọi nơi khác tham chiếu đến nó bằng ID chỉ có nghĩa bên trong cơ sở dữ liệu. Nếu lưu thẳng văn bản, thông tin có ý nghĩa này bị sao chép (*replication*) vào mọi bản ghi dùng đến nó. Biểu diễn như vậy là *phi chuẩn hóa* (*denormalized*).

Lợi ích của ID là bản thân nó không có ý nghĩa với con người nên không bao giờ cần thay đổi: ID có thể giữ nguyên ngay cả khi thông tin mà nó xác định thay đổi. Mọi thông tin có ý nghĩa với con người đều có thể cần sửa đổi trong tương lai. Nếu thông tin đó bị sao chép (*replication*), mọi bản dư thừa đều phải được cập nhật theo. Điều này đòi hỏi thêm mã, thêm thao tác ghi và thêm dung lượng đĩa, đồng thời gây rủi ro không nhất quán, khi một số bản đã cập nhật còn một số thì chưa.

Biểu diễn chuẩn hóa cũng có cái giá: mỗi lần hiển thị một bản ghi chứa ID, ta phải tra cứu thêm để chuyển ID thành thông tin con người đọc được. Trong mô hình dữ liệu quan hệ, việc này được thực hiện bằng *phép nối* (*join*), ví dụ:

```sql
SELECT users.*, regions.region_name
    FROM users
    JOIN regions ON users.region_id = regions.id
    WHERE users.id = 251;
```

Cơ sở dữ liệu tài liệu có thể lưu cả dữ liệu chuẩn hóa lẫn phi chuẩn hóa, nhưng người ta thường gắn nó với phi chuẩn hóa. Một mặt, mô hình dữ liệu JSON giúp dễ dàng thêm các trường phi chuẩn hóa. Mặt khác, nhiều cơ sở dữ liệu tài liệu hỗ trợ phép nối khá yếu, khiến việc dùng biểu diễn chuẩn hóa trở nên bất tiện. Một số cơ sở dữ liệu tài liệu hoàn toàn không hỗ trợ phép nối, nên bạn phải tự làm trong mã ứng dụng: lấy tài liệu chứa ID, rồi thực hiện truy vấn thứ hai để dùng ID tìm tài liệu kia. MongoDB cũng có thể thực hiện phép nối bằng toán tử `$lookup` trong aggregation pipeline:

```mongodb-json
db.users.aggregate([
    { $match: { _id: 251 } },
    { $lookup: {
        from: "regions",
        localField: "region_id",
        foreignField: "_id",
        as: "region"
    } }
])
```

#### Các đánh đổi của chuẩn hóa {#trade-offs-of-normalization}

Trong ví dụ sơ yếu lý lịch, trường `region_id` tham chiếu đến một tập khu vực được chuẩn hóa, còn `organization` (công ty hoặc cơ quan chính phủ nơi làm việc) và `school_name` (trường đã theo học) lại chỉ là chuỗi. Đây là biểu diễn phi chuẩn hóa: nhiều người có thể từng làm ở cùng một công ty, nhưng các bản ghi này không được liên kết với nhau qua ID.

Vậy có nên mô hình hóa tổ chức và trường học thành các thực thể, để hồ sơ tham chiếu ID của chúng thay vì ghi trực tiếp tên? Những lý lẽ ủng hộ việc tham chiếu khu vực bằng ID ở đây cũng áp dụng được. Chẳng hạn, giả sử ngoài tên, ta còn muốn hiển thị logo của trường hoặc công ty:

* Với biểu diễn phi chuẩn hóa, hồ sơ của mỗi người phải chứa URL của ảnh logo. Cách này khiến tài liệu JSON tự đóng gói đầy đủ, nhưng nếu logo thay đổi, bạn phải tìm mọi nơi xuất hiện URL cũ để cập nhật từng chỗ, rất phiền toái [^9].
* Với biểu diễn chuẩn hóa, có thể tạo một thực thể đại diện cho tổ chức hoặc trường học, lưu tên, URL logo, có thể thêm phần giới thiệu, bảng tin, v.v. một lần duy nhất. Mọi sơ yếu lý lịch nhắc đến tổ chức đó chỉ tham chiếu ID của nó, nên việc cập nhật logo trở nên dễ dàng.

Nói chung, dữ liệu chuẩn hóa ghi nhanh hơn (vì chỉ có một bản) nhưng truy vấn chậm hơn (vì cần phép nối). Dữ liệu phi chuẩn hóa thường đọc nhanh hơn (ít phép nối hơn) nhưng ghi tốn kém hơn (phải cập nhật nhiều bản, và tốn thêm dung lượng đĩa). Có thể coi phi chuẩn hóa là một dạng dữ liệu dẫn xuất (xem [“Hệ thống ghi nhận gốc và dữ liệu dẫn xuất”](/ch1#sec_introduction_derived)), vì bạn phải thiết lập quy trình để cập nhật các bản sao dư thừa của dữ liệu.

Ngoài chi phí cập nhật, còn phải xét xem cơ sở dữ liệu có giữ được tính nhất quán hay không khi tiến trình gặp sự cố giữa chừng lúc cập nhật. Cơ sở dữ liệu hỗ trợ giao dịch nguyên tử (xem [“Tính nguyên tử”](/ch8#sec_transactions_acid_atomicity)) duy trì tính nhất quán dễ hơn, nhưng không phải cơ sở dữ liệu nào cũng đảm bảo tính nguyên tử cho thao tác trải qua nhiều tài liệu. Cũng có thể dùng xử lý luồng để đảm bảo tính nhất quán. Ta sẽ bàn cách này ở [“Giữ các hệ thống đồng bộ”](/ch12#sec_stream_sync).

Chuẩn hóa thường phù hợp hơn với hệ thống OLTP, nơi cả đọc lẫn ghi đều phải nhanh. Hệ thống phân tích thường hợp với dữ liệu phi chuẩn hóa hơn, vì chúng cập nhật theo lô và mối quan tâm hàng đầu là hiệu năng của truy vấn chỉ đọc. Ngoài ra, ở hệ thống quy mô vừa và nhỏ, mô hình dữ liệu chuẩn hóa thường là lựa chọn tốt: không cần lo giữ nhất quán giữa nhiều bản sao, và chi phí thực hiện phép nối cũng chấp nhận được. Tuy nhiên, ở quy mô cực lớn, chi phí của phép nối có thể trở thành vấn đề.

#### Phi chuẩn hóa trong nghiên cứu tình huống mạng xã hội {#denormalization-in-the-social-networking-case-study}

Trong [“Nghiên cứu tình huống: dòng thời gian trang chủ của mạng xã hội”](/ch2#sec_introduction_twitter), ta đã so sánh biểu diễn chuẩn hóa ({{< xref fig="2-1" page="/ch2" anchor="fig_twitter_relational" >}}Hình 2-1{{< /xref >}}) với biểu diễn phi chuẩn hóa (dòng thời gian được tính trước và vật chất hóa). Trong ví dụ đó, phép nối `posts` với `follows` quá tốn kém, nên dòng thời gian vật chất hóa đóng vai trò bộ nhớ đệm cho kết quả phép nối. Việc fan-out bài đăng mới đến dòng thời gian của người theo dõi chính là cách duy trì tính nhất quán của biểu diễn phi chuẩn hóa này.

Tuy nhiên, dòng thời gian vật chất hóa của X (trước đây là Twitter) thực ra không lưu nội dung của từng bài đăng. Mỗi mục chỉ lưu ID bài đăng, ID người đăng, cùng một ít thông tin bổ sung để nhận biết bài đăng lại và trả lời [^11]. Nói cách khác, nó gần như là kết quả được tính trước của truy vấn sau:

```sql
SELECT posts.id, posts.sender_id 
    FROM posts
    JOIN follows ON posts.sender_id = follows.followee_id
    WHERE follows.follower_id = current_user
    ORDER BY posts.timestamp DESC
    LIMIT 1000
```

Vì vậy, mỗi lần đọc dòng thời gian, dịch vụ vẫn phải thực hiện hai phép nối: tra theo ID bài đăng để lấy nội dung cùng các thống kê như số lượt thích và số lượt trả lời, và tra theo ID người đăng để lấy tên người dùng, ảnh đại diện và thông tin hồ sơ khác. Quá trình chuyển ID thành thông tin con người đọc được này gọi là *bổ sung ID* (*hydrating IDs*), về bản chất là thực hiện phép nối ngay trong mã ứng dụng [^11].

Dòng thời gian tính trước chỉ lưu ID vì dữ liệu mà ID trỏ tới thay đổi rất nhanh: số lượt thích và trả lời của bài đăng phổ biến có thể thay đổi nhiều lần mỗi giây, và một số người dùng thường xuyên đổi tên người dùng hoặc ảnh đại diện. Dòng thời gian khi hiển thị phải cho thấy số lượt thích và ảnh đại diện mới nhất, nên phi chuẩn hóa những thông tin này vào dòng thời gian vật chất hóa là không hợp lý, và còn làm chi phí lưu trữ tăng mạnh.

Ví dụ này cho thấy việc phải thực hiện phép nối khi đọc dữ liệu không cản trở ta xây dựng dịch vụ hiệu năng cao và có khả năng mở rộng, như một số quan điểm vẫn nói. Bổ sung ID bài đăng và ID người dùng thực ra khá dễ mở rộng: công việc này dễ song song hóa, và chi phí không phụ thuộc vào số tài khoản bạn theo dõi hay số người theo dõi bạn.

Khi cần quyết định có nên phi chuẩn hóa một phần dữ liệu trong ứng dụng hay không, ca nghiên cứu mạng xã hội cho thấy đáp án không hiển nhiên: giải pháp mở rộng tốt nhất có thể là phi chuẩn hóa một số dữ liệu và giữ một số khác ở dạng chuẩn hóa. Bạn phải cân nhắc kỹ tần suất thay đổi của thông tin cùng chi phí đọc và ghi, mà chi phí này lại có thể bị chi phối bởi các trường hợp cực đoan, chẳng hạn người dùng mạng xã hội có số người theo dõi hoặc số người được theo dõi lớn bất thường. Chuẩn hóa và phi chuẩn hóa tự thân không tốt cũng không xấu. Chúng chỉ là sự đánh đổi giữa hiệu năng đọc ghi và chi phí triển khai.

### Quan hệ nhiều-một và nhiều-nhiều {#sec_datamodels_many_to_many}

Trong {{< xref fig="3-1" page="/ch3" anchor="fig_obama_relational" >}}Hình 3-1{{< /xref >}}, `positions` và `education` là quan hệ một-nhiều (hay một-vài): một bản sơ yếu lý lịch có nhiều vị trí công tác, nhưng mỗi vị trí chỉ thuộc về một bản sơ yếu lý lịch. Ngược lại, trường `region_id` biểu diễn quan hệ *nhiều-một* (*many-to-one*): nhiều người sống ở cùng một khu vực, và ta giả định tại mỗi thời điểm mỗi người chỉ sống ở một khu vực.

Nếu tiếp tục mô hình hóa tổ chức và trường học thành thực thể, để sơ yếu lý lịch tham chiếu chúng bằng ID, thì sẽ xuất hiện quan hệ *nhiều-nhiều* (*many-to-many*): một người từng làm ở nhiều tổ chức, và một tổ chức có nhiều nhân viên hiện tại hoặc cũ. Trong mô hình quan hệ, loại quan hệ này thường được biểu diễn bằng *bảng liên kết* (*associative table*, còn gọi là *bảng nối*, *join table*), như trong {{< xref fig="3-3" page="/ch3" anchor="fig_datamodels_m2m_rel" >}}Hình 3-3{{< /xref >}}: mỗi vị trí công tác liên kết một ID người dùng với một ID tổ chức.

{{< fig num="3-3" id="fig_datamodels_m2m_rel" src="/fig/ddia_0303.png" caption="Quan hệ nhiều-nhiều trong mô hình quan hệ." class="ddia-figure ddia-figure--wide" width="1772" height="745" />}}

Quan hệ nhiều-một và nhiều-nhiều khó nhét vào một tài liệu JSON tự đóng gói. Chúng phù hợp hơn với biểu diễn chuẩn hóa. {{< xref eg="3-2" page="/ch3" anchor="fig_datamodels_m2m_json" >}}Ví dụ 3-2{{< /xref >}} đưa ra một cách làm trong mô hình tài liệu, còn {{< xref fig="3-4" page="/ch3" anchor="fig_datamodels_many_to_many" >}}Hình 3-4{{< /xref >}} minh họa bằng sơ đồ: dữ liệu trong mỗi khung nét đứt có thể tạo thành một tài liệu, nhưng liên kết đến tổ chức và trường học tốt nhất nên biểu diễn dưới dạng tham chiếu đến các tài liệu khác.

{{< eg num="3-2" id="fig_datamodels_m2m_json" caption="Sơ yếu lý lịch tham chiếu tổ chức bằng ID" >}}
```json
{
    "user_id": 251,
    "first_name": "Barack",
    "last_name": "Obama",
    "positions": [
        {"start": 2009, "end": 2017, "job_title": "President", "org_id": 513},
        {"start": 2005, "end": 2008, "job_title": "US Senator (D-IL)", "org_id": 514}
    ],
    ...
}
```
{{< /eg >}}

{{< fig num="3-4" id="fig_datamodels_many_to_many" src="/fig/ddia_0304.png" caption="Quan hệ nhiều-nhiều trong mô hình tài liệu; dữ liệu trong mỗi khung nét đứt có thể tạo thành một tài liệu." class="ddia-figure ddia-figure--standard" width="1772" height="1121" />}}

Quan hệ nhiều-nhiều thường cần truy vấn theo “hai chiều”: vừa tìm mọi tổ chức mà một người từng làm việc, vừa tìm mọi người từng làm việc tại một tổ chức. Một cách là lưu tham chiếu ID ở cả hai đầu quan hệ: sơ yếu lý lịch liệt kê ID các tổ chức người đó từng làm, còn tài liệu tổ chức liệt kê ID các sơ yếu lý lịch nhắc đến nó. Vì cùng một quan hệ được lưu hai lần, đây là biểu diễn phi chuẩn hóa, và hai phía có thể không nhất quán với nhau.

Biểu diễn chuẩn hóa chỉ lưu quan hệ ở một nơi, rồi dựa vào *chỉ mục phụ* (*secondary index*) (sẽ bàn ở [Chương 4](/ch4#ch_storage)) để truy vấn hiệu quả theo cả hai chiều. Với schema quan hệ trong {{< xref fig="3-3" page="/ch3" anchor="fig_datamodels_m2m_rel" >}}Hình 3-3{{< /xref >}}, ta có thể cho cơ sở dữ liệu tạo chỉ mục (*index*) riêng cho cột `user_id` và cột `org_id` của bảng `positions`.

Trong mô hình tài liệu ở {{< xref eg="3-2" page="/ch3" anchor="fig_datamodels_m2m_json" >}}Ví dụ 3-2{{< /xref >}}, cơ sở dữ liệu thì cần lập chỉ mục (*index*) trên trường `org_id` của các đối tượng nằm trong mảng `positions`. Nhiều cơ sở dữ liệu tài liệu, cũng như các cơ sở dữ liệu quan hệ hỗ trợ JSON, đều có thể lập loại chỉ mục (*index*) này trên các giá trị bên trong tài liệu.

### Star schema và snowflake schema: các schema cho phân tích {#sec_datamodels_analytics}

Kho dữ liệu (xem [“Kho dữ liệu”](/ch1#sec_introduction_dwh)) thường dùng mô hình quan hệ, với cấu trúc bảng theo một số quy ước được áp dụng rộng rãi: *star schema*, *snowflake schema*, *mô hình hóa theo chiều* (*dimensional modeling*) [^12], và *một bảng lớn* (*one big table*, OBT). Các cấu trúc này được tối ưu cho nhu cầu của các nhà phân tích nghiệp vụ, còn quy trình ETL chịu trách nhiệm chuyển dữ liệu từ các hệ thống giao dịch sang schema đó.

Schema mẫu trong {{< xref fig="3-5" page="/ch3" anchor="fig_dwh_schema" >}}Hình 3-5{{< /xref >}} có thể xuất hiện trong kho dữ liệu của một nhà bán lẻ thực phẩm. Ở trung tâm của schema là cái gọi là *bảng sự kiện* (*fact table*, trong ví dụ này tên là `fact_sales`). Mỗi hàng của bảng sự kiện biểu diễn một sự kiện xảy ra tại một thời điểm cụ thể. Ở đây, mỗi hàng là một lần khách hàng mua một sản phẩm. Nếu phân tích lưu lượng truy cập web thay vì doanh số bán lẻ, mỗi hàng có thể là một lượt xem trang hoặc một lần nhấp chuột của người dùng.

{{< fig num="3-5" id="fig_dwh_schema" src="/fig/ddia_0305.png" caption="Ví dụ về star schema dùng trong kho dữ liệu." class="ddia-figure ddia-figure--standard" width="2658" height="2223" />}}

Mỗi sự kiện thường được ghi lại như một event riêng lẻ, vì cách này giữ được tối đa tính linh hoạt cho các phân tích sau này. Tuy nhiên, điều đó cũng có nghĩa là bảng sự kiện có thể lớn đến mức cực đoan. Kho dữ liệu của một doanh nghiệp lớn có thể chứa nhiều petabyte lịch sử giao dịch, phần lớn nằm trong các bảng sự kiện.

Một số cột của bảng sự kiện là thuộc tính, chẳng hạn giá bán của sản phẩm và chi phí mua từ nhà cung cấp (từ đó có thể tính biên lợi nhuận). Các cột khác là tham chiếu khóa ngoại tới những bảng khác, gọi là *bảng chiều* (*dimension table*). Vì mỗi hàng của bảng sự kiện biểu diễn một sự kiện, nên các chiều biểu diễn ai, cái gì, ở đâu, khi nào, như thế nào và tại sao của sự kiện đó.

Chẳng hạn, trong {{< xref fig="3-5" page="/ch3" anchor="fig_dwh_schema" >}}Hình 3-5{{< /xref >}}, một chiều là sản phẩm được bán. Mỗi hàng của bảng `dim_product` là một loại sản phẩm đang bán, gồm mã đơn vị lưu kho (SKU), mô tả sản phẩm, tên thương hiệu, danh mục, hàm lượng chất béo, kích thước bao bì, v.v. Mỗi hàng của bảng `fact_sales` dùng một khóa ngoại để cho biết giao dịch đó đã bán sản phẩm nào. Truy vấn thường phải join nhiều bảng chiều.

Ngay cả ngày và giờ cũng thường được biểu diễn bằng bảng chiều, để mã hóa thêm thông tin như ngày lễ, cho phép truy vấn phân biệt doanh số của ngày nghỉ lễ với ngày thường.

{{< xref fig="3-5" page="/ch3" anchor="fig_dwh_schema" >}}Hình 3-5{{< /xref >}} là một ví dụ về star schema. Tên gọi này bắt nguồn từ hình dạng khi trực quan hóa quan hệ giữa các bảng: bảng sự kiện nằm ở giữa, được bao quanh bởi các bảng chiều; các đường nối giữa các bảng tỏa ra như tia sáng của ngôi sao.

Một biến thể của mẫu này gọi là *snowflake schema*, trong đó các chiều được phân rã tiếp thành các chiều con. Chẳng hạn, có thể tạo bảng riêng cho thương hiệu và danh mục sản phẩm, để mỗi hàng của `dim_product` tham chiếu thương hiệu và danh mục bằng khóa ngoại, thay vì lưu chúng dưới dạng chuỗi ngay trong bảng `dim_product`. Snowflake schema được chuẩn hóa nhiều hơn star schema, nhưng star schema thường được ưa chuộng hơn vì các nhà phân tích dùng đơn giản hơn [^12].

Các bảng trong kho dữ liệu điển hình thường rất rộng: bảng sự kiện hay có hơn 100 cột, đôi khi lên đến hàng trăm cột. Các bảng chiều cũng có thể rất rộng, vì chúng gom mọi siêu dữ liệu có thể liên quan đến phân tích. Chẳng hạn, bảng `dim_store` có thể ghi lại mỗi cửa hàng cung cấp dịch vụ nào, có tiệm bánh trong cửa hàng hay không, diện tích mặt bằng, ngày khai trương lần đầu, lần cải tạo gần nhất, khoảng cách đến đường cao tốc gần nhất, v.v.

Star schema và snowflake schema chủ yếu gồm các quan hệ nhiều-một, chẳng hạn nhiều giao dịch bán hàng cùng thuộc một sản phẩm, một cửa hàng. Các quan hệ này thể hiện qua khóa ngoại từ bảng sự kiện tới bảng chiều, hoặc từ bảng chiều tới bảng chiều con. Về nguyên tắc cũng có thể có các loại quan hệ khác, nhưng để đơn giản hóa truy vấn, chúng thường được phi chuẩn hóa. Chẳng hạn, khi khách hàng mua nhiều sản phẩm cùng lúc, giao dịch nhiều mặt hàng đó không được biểu diễn tường minh. Bảng sự kiện chỉ lưu một hàng cho mỗi mặt hàng, và các sự kiện này tình cờ có cùng ID khách hàng, ID cửa hàng và dấu thời gian.

Một số schema của kho dữ liệu còn đi xa hơn: bỏ hẳn các bảng chiều, đưa thông tin chiều vào các cột phi chuẩn hóa ngay trong bảng sự kiện — về thực chất là tính trước phép join giữa bảng sự kiện và các bảng chiều. Cách này gọi là *một bảng lớn* (OBT), tuy tốn thêm dung lượng lưu trữ, đôi khi nó giúp truy vấn nhanh hơn [^13].

Trong ngữ cảnh phân tích, việc phi chuẩn hóa như vậy thường không thành vấn đề, vì dữ liệu thường là bản ghi lịch sử không bao giờ thay đổi (trừ khi sửa lỗi thỉnh thoảng). Các vấn đề về tính nhất quán dữ liệu và chi phí ghi mà phi chuẩn hóa gây ra trong hệ thống OLTP không cấp bách đến thế ở hệ thống phân tích.

### Khi nào dùng mô hình nào {#sec_datamodels_document_summary}

Các lập luận chính ủng hộ mô hình dữ liệu tài liệu là tính linh hoạt của schema, hiệu năng tốt hơn nhờ tính cục bộ, và — với một số ứng dụng — việc nó gần với mô hình đối tượng mà ứng dụng sử dụng. Mô hình quan hệ đáp lại bằng khả năng hỗ trợ tốt hơn cho join, cũng như các quan hệ nhiều-một và nhiều-nhiều. Ta sẽ lần lượt xem xét chi tiết từng lập luận.

Nếu dữ liệu trong ứng dụng có cấu trúc giống tài liệu (tức là một cây các quan hệ một-nhiều, thường được tải cả cây một lần), thì dùng mô hình tài liệu có thể là ý hay. Việc *shredding* cấu trúc kiểu tài liệu vào nhiều bảng (như `positions`, `education` và `contact_info` trong {{< xref fig="3-1" page="/ch3" anchor="fig_obama_relational" >}}Hình 3-1{{< /xref >}}) có thể dẫn đến schema rườm rà và mã ứng dụng phức tạp không cần thiết.

Mô hình tài liệu có những hạn chế nhất định: chẳng hạn, bạn không thể tham chiếu trực tiếp một mục lồng bên trong tài liệu, mà phải nói kiểu “mục thứ hai trong danh sách vị trí công tác của người dùng 251”. Nếu thực sự cần tham chiếu các mục lồng nhau, mô hình quan hệ phù hợp hơn, vì mọi mục đều có thể được tham chiếu trực tiếp bằng ID của chính nó.

Một số ứng dụng cho phép người dùng tự sắp xếp thứ tự các mục. Chẳng hạn, trong danh sách việc cần làm hoặc trình theo dõi issue, người dùng có thể kéo thả các tác vụ để đổi thứ tự. Mô hình tài liệu rất phù hợp với các ứng dụng này, chỉ cần lưu các mục (hoặc ID của chúng) theo thứ tự trong một mảng JSON. Cơ sở dữ liệu quan hệ không có cách chuẩn để biểu diễn loại danh sách có thể đổi thứ tự này, mà phải dùng nhiều mẹo: sắp xếp theo một cột số nguyên (chèn vào giữa thì phải đánh số lại), dùng danh sách liên kết các ID, hoặc dùng *fractional indexing* [^14] [^15] [^16].

#### Tính linh hoạt của schema trong mô hình tài liệu {#sec_datamodels_schema_flexibility}

Hầu hết cơ sở dữ liệu tài liệu, cũng như phần hỗ trợ JSON trong các cơ sở dữ liệu quan hệ, không ép dữ liệu trong tài liệu phải theo một schema nào. Phần hỗ trợ XML của cơ sở dữ liệu quan hệ thường đi kèm khả năng kiểm tra hợp lệ theo schema (tùy chọn). Không có schema nghĩa là có thể thêm khóa và giá trị tùy ý vào tài liệu, còn khi đọc, client không có gì đảm bảo về việc tài liệu chứa những trường nào.

Cơ sở dữ liệu tài liệu đôi khi được gọi là *không có schema* (*schemaless*), nhưng cách gọi này gây hiểu lầm, vì mã đọc dữ liệu thường giả định một cấu trúc nào đó — tức là có một schema ngầm định, chỉ là cơ sở dữ liệu không thực thi nó [^17]. Thuật ngữ chính xác hơn là *schema-on-read* (cấu trúc dữ liệu là ngầm định và chỉ được diễn giải khi đọc), đối lập với *schema-on-write* (cách làm truyền thống của cơ sở dữ liệu quan hệ: schema là tường minh và cơ sở dữ liệu đảm bảo mọi dữ liệu được ghi đều tuân theo nó) [^18].

Schema-on-read giống với kiểm tra kiểu động (lúc chạy) trong ngôn ngữ lập trình, còn schema-on-write giống với kiểm tra kiểu tĩnh (lúc biên dịch). Cũng như việc kiểu tĩnh hay kiểu động tốt hơn là chủ đề tranh cãi dai dẳng [^19], việc cơ sở dữ liệu có nên thực thi schema hay không cũng là vấn đề quan điểm, và nhìn chung không có đúng sai tuyệt đối.

Sự khác biệt giữa hai cách tiếp cận đặc biệt rõ khi ứng dụng muốn thay đổi định dạng dữ liệu. Chẳng hạn, giả sử trước đây bạn lưu họ tên đầy đủ của mỗi người dùng trong một trường, nay muốn lưu tên và họ riêng [^20]. Trong cơ sở dữ liệu tài liệu, bạn chỉ cần bắt đầu ghi các tài liệu có trường mới, rồi thêm mã trong ứng dụng để xử lý các tài liệu cũ. Ví dụ:

```mongodb-json
if (user && user.name && !user.first_name) {
    // Các tài liệu được ghi trước ngày 8 tháng 12 năm 2023 không có first_name
    user.first_name = user.name.split(" ")[0];
}
```

Nhược điểm của cách này là mọi phần của ứng dụng đọc từ cơ sở dữ liệu từ nay đều phải xử lý được các tài liệu định dạng cũ, có thể đã được ghi từ rất lâu. Ngược lại, trong cơ sở dữ liệu schema-on-write, thường sẽ thực hiện một *migration* như sau:

```sql
ALTER TABLE users ADD COLUMN first_name text DEFAULT NULL;
UPDATE users SET first_name = split_part(name, ' ', 1); -- PostgreSQL
UPDATE users SET first_name = substring_index(name, ' ', 1); -- MySQL
```

Trong hầu hết cơ sở dữ liệu quan hệ, việc thêm một cột có giá trị mặc định là nhanh và an toàn, ngay cả với bảng lớn. Tuy nhiên, chạy `UPDATE` trên bảng lớn có thể rất chậm, vì mọi hàng đều phải được ghi lại. Các thao tác schema khác (chẳng hạn đổi kiểu dữ liệu của một cột) cũng thường yêu cầu sao chép (*replication*) toàn bộ bảng.

Có nhiều công cụ cho phép thực hiện các thay đổi schema như vậy ở chế độ nền mà không cần dừng hệ thống [^21] [^22] [^23] [^24], nhưng vận hành migration kiểu này trên cơ sở dữ liệu lớn vẫn là thách thức. Để tránh migration phức tạp, bạn có thể chỉ thêm nhanh một cột `first_name` với giá trị mặc định `NULL`, rồi điền giá trị lúc đọc, giống như khi dùng cơ sở dữ liệu tài liệu.

Schema-on-read có lợi thế hơn khi các mục trong một tập hợp, vì lý do nào đó, không có cùng cấu trúc (tức dữ liệu không đồng nhất). Chẳng hạn khi:

* Có nhiều loại đối tượng khác nhau, và việc đưa mỗi loại vào một bảng riêng là không thực tế.
* Cấu trúc dữ liệu do các hệ thống bên ngoài quyết định, bạn không kiểm soát được và chúng có thể thay đổi bất cứ lúc nào.

Trong những trường hợp đó, schema có thể hại nhiều hơn lợi, và tài liệu không có schema có thể là mô hình dữ liệu tự nhiên hơn. Nhưng nếu mọi bản ghi đều phải có cùng cấu trúc, schema là cơ chế hữu hiệu để ghi lại và thực thi cấu trúc đó. Ta sẽ bàn kỹ hơn về schema và sự tiến hóa của schema ở [Chương 5](/ch5#ch_encoding).

#### Tính cục bộ của dữ liệu khi đọc và ghi {#sec_datamodels_document_locality}

Tài liệu thường được lưu dưới dạng một chuỗi liên tục duy nhất, mã hóa thành JSON, XML hoặc biến thể nhị phân của chúng (như BSON của MongoDB). Nếu ứng dụng thường xuyên cần truy cập toàn bộ tài liệu (ví dụ để hiển thị lên trang web), *tính cục bộ của lưu trữ* (*storage locality*) này mang lại lợi thế về hiệu năng. Nếu dữ liệu bị phân tán trên nhiều bảng như trong {{< xref fig="3-1" page="/ch3" anchor="fig_obama_relational" >}}Hình 3-1{{< /xref >}}, thì cần nhiều lần tra cứu chỉ mục (*index*) mới lấy đủ dữ liệu, có thể gây thêm nhiều lần seek trên đĩa và tốn thời gian hơn.

Lợi thế về tính cục bộ chỉ áp dụng khi bạn cần một phần đáng kể của tài liệu cùng lúc. Ngay cả khi chỉ truy cập một phần nhỏ của tài liệu lớn, cơ sở dữ liệu thường vẫn phải tải toàn bộ tài liệu, gây lãng phí. Khi cập nhật thì thường phải ghi lại cả tài liệu. Vì vậy, thường nên giữ tài liệu nhỏ và tránh cập nhật nhỏ lẻ thường xuyên lên tài liệu.

Tuy nhiên, việc lưu dữ liệu liên quan cạnh nhau vì mục đích cục bộ không phải đặc quyền của mô hình tài liệu. Chẳng hạn, cơ sở dữ liệu Spanner của Google cũng cung cấp thuộc tính cục bộ tương tự trong mô hình quan hệ, cho phép schema khai báo rằng các hàng của một bảng được xen kẽ (lồng) trong bảng cha [^25]. Oracle cũng cung cấp khả năng tương tự qua tính năng *multi-table index cluster table* [^26]. Mô hình dữ liệu *wide-column*, do Google Bigtable phổ biến và được các hệ thống như HBase và Accumulo áp dụng, dùng khái niệm *column family* để đạt mục tiêu quản lý tính cục bộ tương tự [^27].

#### Ngôn ngữ truy vấn cho tài liệu {#query-languages-for-documents}

Một điểm khác biệt nữa giữa cơ sở dữ liệu quan hệ và cơ sở dữ liệu tài liệu là ngôn ngữ hoặc API dùng để truy vấn. Hầu hết cơ sở dữ liệu quan hệ dùng SQL, còn cơ sở dữ liệu tài liệu thì muôn hình muôn vẻ: có loại chỉ cho truy cập khóa-giá trị theo khóa chính, có loại còn cung cấp chỉ mục phụ (*secondary index*) để truy vấn các giá trị bên trong tài liệu, và có loại có ngôn ngữ truy vấn phong phú.

Cơ sở dữ liệu XML thường dùng XQuery và XPath. Chúng hỗ trợ truy vấn phức tạp, kể cả join giữa các tài liệu, và có thể định dạng kết quả thành XML [^28]. JSON Pointer [^29] và JSONPath [^30] cung cấp cho JSON các chức năng tương đương XPath.

Aggregation pipeline của MongoDB là một ngôn ngữ truy vấn cho các tập hợp tài liệu JSON. Ta đã thấy toán tử `$lookup` của nó dùng cho join trong [“Chuẩn hóa, phi chuẩn hóa và phép nối”](/ch3#sec_datamodels_normalization).

Hãy xem thêm một ví dụ để cảm nhận loại ngôn ngữ này, lần này là phép tổng hợp (aggregation), vốn đặc biệt phổ biến trong phân tích. Giả sử bạn là nhà sinh vật biển, mỗi lần thấy một loài động vật ngoài biển lại thêm một bản ghi quan sát vào cơ sở dữ liệu. Bây giờ bạn muốn lập báo cáo cho biết mỗi tháng đã quan sát được bao nhiêu con cá mập. Trong PostgreSQL, có thể diễn đạt truy vấn như sau:

```sql
SELECT date_trunc('month', observation_timestamp) AS observation_month, ❶ 
    sum(num_animals) AS total_animals
FROM observations
WHERE family = 'Sharks'
GROUP BY observation_month;
```

❶: Hàm `date_trunc('month', timestamp)` xác định tháng dương lịch chứa `timestamp`, rồi trả về một dấu thời gian khác biểu diễn thời điểm bắt đầu của tháng đó. Nói cách khác, nó làm tròn xuống dấu thời gian về tháng gần nhất.

Truy vấn này trước hết lọc các bản ghi quan sát để chỉ giữ các loài thuộc họ cá mập, rồi nhóm theo tháng dương lịch của lần quan sát, và cuối cùng cộng số lượng động vật trong mọi bản ghi quan sát của tháng đó. Cùng truy vấn này có thể được biểu diễn bằng aggregation pipeline của MongoDB như sau:

```mongodb-json
db.observations.aggregate([
    { $match: { family: "Sharks" } },
    { $group: {
    _id: {
        year: { $year: "$observationTimestamp" },
        month: { $month: "$observationTimestamp" }
    },
    totalAnimals: { $sum: "$numAnimals" }
    } }
]);
```

Ngôn ngữ aggregation pipeline có sức biểu đạt tương đương một tập con của SQL, nhưng dùng cú pháp dựa trên JSON thay vì kiểu câu gần với tiếng Anh của SQL. Sự khác biệt này có lẽ chỉ là vấn đề sở thích.

#### Sự hội tụ của cơ sở dữ liệu tài liệu và quan hệ {#convergence-of-document-and-relational-databases}

Cơ sở dữ liệu tài liệu và cơ sở dữ liệu quan hệ ban đầu có cách quản lý dữ liệu rất khác nhau, nhưng theo thời gian chúng ngày càng giống nhau [^31]. Cơ sở dữ liệu quan hệ đã bổ sung hỗ trợ kiểu JSON và các toán tử truy vấn, cũng như khả năng lập chỉ mục (*index*) trên thuộc tính bên trong tài liệu, còn các cơ sở dữ liệu tài liệu như MongoDB, Couchbase và RethinkDB đã bổ sung join, chỉ mục phụ (*secondary index*) và ngôn ngữ truy vấn khai báo.

Sự hội tụ này có lợi cho lập trình viên ứng dụng, vì mô hình quan hệ và mô hình tài liệu phát huy tốt nhất khi có thể kết hợp trong cùng một cơ sở dữ liệu. Nhiều cơ sở dữ liệu tài liệu cần tham chiếu các tài liệu khác theo kiểu quan hệ, và nhiều cơ sở dữ liệu quan hệ cũng có những phần được hưởng lợi từ tính linh hoạt của schema. Sự kết hợp giữa mô hình quan hệ và mô hình tài liệu là một tổ hợp mạnh mẽ.


> [!NOTE]
> Mô tả ban đầu của Codd về mô hình quan hệ [^3] thực ra cho phép các cấu trúc giống JSON xuất hiện trong schema quan hệ, mà ông gọi là *miền không đơn giản* (*nonsimple domain*). Ý tưởng là giá trị trong một hàng không nhất thiết chỉ là kiểu dữ liệu nguyên thủy như số hay chuỗi, mà còn có thể là một quan hệ (bảng) lồng bên trong, nhờ đó có thể dùng cấu trúc cây lồng nhau tùy ý làm một giá trị. Điều này rất giống phần hỗ trợ JSON hoặc XML được thêm vào SQL hơn ba mươi năm sau.



## Mô hình dữ liệu đồ thị {#sec_datamodels_graph}

Như đã thấy, loại quan hệ là một đặc điểm quan trọng để phân biệt các mô hình dữ liệu. Nếu quan hệ trong ứng dụng chủ yếu là một-nhiều (dữ liệu có cấu trúc cây) và giữa các bản ghi hầu như không có quan hệ nào khác, thì mô hình tài liệu là phù hợp.

Nhưng nếu quan hệ nhiều-nhiều rất phổ biến trong dữ liệu thì sao? Mô hình quan hệ xử lý được các quan hệ nhiều-nhiều đơn giản, nhưng khi các kết nối giữa dữ liệu ngày càng phức tạp, việc mô hình hóa dữ liệu dưới dạng đồ thị trở nên tự nhiên hơn.

Một đồ thị gồm hai loại đối tượng: *đỉnh* (*vertices*, còn gọi là *nút* — *nodes* — hoặc *thực thể* — *entities*) và *cạnh* (*edges*, còn gọi là *quan hệ* — *relationships* — hoặc *cung* — *arcs*). Nhiều loại dữ liệu có thể được mô hình hóa thành đồ thị, ví dụ điển hình gồm:

Đồ thị xã hội
: Đỉnh là con người, cạnh cho biết những ai quen biết nhau.

Đồ thị web
: Đỉnh là các trang web, cạnh là các liên kết HTML trỏ tới trang khác.

Mạng lưới đường bộ hoặc đường sắt
: Đỉnh là các giao lộ, cạnh là các tuyến đường bộ hoặc đường sắt nối giữa chúng.

Nhiều thuật toán nổi tiếng có thể áp dụng trên các đồ thị này. Chẳng hạn, ứng dụng bản đồ tìm đường đi ngắn nhất giữa hai điểm trong mạng lưới đường bộ; PageRank có thể dùng trên đồ thị web để xác định mức độ phổ biến của một trang web, từ đó quyết định thứ hạng của nó trong kết quả tìm kiếm [^32].

Đồ thị có thể được biểu diễn theo vài cách khác nhau. Trong mô hình *danh sách kề* (*adjacency list*), mỗi đỉnh lưu ID của các đỉnh lân cận cách nó một cạnh. Một cách khác là *ma trận kề* (*adjacency matrix*): một mảng hai chiều trong đó mỗi hàng và mỗi cột ứng với một đỉnh. Giá trị bằng 0 khi không có cạnh giữa đỉnh hàng và đỉnh cột, và bằng 1 khi có cạnh. Danh sách kề thích hợp cho việc duyệt đồ thị, còn ma trận kề thích hợp cho học máy (xem [“Dataframe, ma trận và mảng”](/ch3#sec_datamodels_dataframes)).

Trong các ví dụ vừa nêu, mọi đỉnh trong đồ thị đều biểu diễn cùng một loại thứ: lần lượt là con người, trang web hoặc giao lộ. Tuy nhiên, đồ thị không giới hạn ở dữ liệu *đồng nhất* (*homogeneous*) như vậy: một công dụng mạnh mẽ không kém của đồ thị là lưu trữ các loại đối tượng hoàn toàn khác nhau một cách nhất quán trong cùng một cơ sở dữ liệu. Ví dụ:

* Facebook duy trì một đồ thị với nhiều loại đỉnh và cạnh khác nhau: đỉnh biểu diễn con người, địa điểm, sự kiện, lượt check-in và bình luận của người dùng. Cạnh biểu diễn ai là bạn của ai, lượt check-in diễn ra ở đâu, ai đã bình luận bài đăng nào, ai tham dự sự kiện nào, v.v. [^33].
* Công cụ tìm kiếm dùng đồ thị tri thức (knowledge graph) để ghi lại các sự kiện về những thực thể thường xuất hiện trong truy vấn, như tổ chức, con người, địa điểm [^34]. Thông tin này đến từ việc thu thập dữ liệu website và phân tích văn bản. Các trang như Wikidata cũng công bố dữ liệu đồ thị ở dạng có cấu trúc.

Có vài cách khác nhau nhưng liên quan với nhau để tổ chức và truy vấn dữ liệu trong đồ thị. Phần này sẽ thảo luận mô hình *property graph* (được hiện thực trong các hệ thống như Neo4j, Memgraph, KùzuDB [^35] [^36]) và mô hình *triple store* (được hiện thực trong các hệ thống như Datomic, AllegroGraph, Blazegraph). Sức biểu đạt của hai mô hình khá gần nhau, và một số cơ sở dữ liệu đồ thị như Amazon Neptune hỗ trợ cả hai.

Ta cũng sẽ giới thiệu bốn ngôn ngữ truy vấn đồ thị (Cypher, SPARQL, Datalog và GraphQL), cùng khả năng hỗ trợ truy vấn đồ thị của SQL. Còn có các ngôn ngữ truy vấn đồ thị khác như Gremlin [^37], nhưng mấy ngôn ngữ được chọn ở đây đã đủ cho một bức tranh toàn cảnh mang tính đại diện.

Để minh họa các ngôn ngữ và mô hình này, phần này dùng {{< xref fig="3-6" page="/ch3" anchor="fig_datamodels_graph" >}}Hình 3-6{{< /xref >}} làm ví dụ xuyên suốt. Nó có thể lấy từ một mạng xã hội hoặc cơ sở dữ liệu phả hệ: đồ thị có hai người, Lucy đến từ Idaho và Alain đến từ Saint-Lô, Pháp. Họ đã kết hôn và hiện sống ở London. Mỗi người và mỗi địa điểm được biểu diễn là một đỉnh, còn quan hệ giữa chúng là các cạnh. Ví dụ này sẽ cho thấy một số truy vấn dễ thực hiện trong cơ sở dữ liệu đồ thị nhưng khó diễn đạt trong các mô hình khác.

{{< fig num="3-6" id="fig_datamodels_graph" src="/fig/ddia_0306.png" caption="Ví dụ về dữ liệu có cấu trúc đồ thị (hình chữ nhật là đỉnh, mũi tên là cạnh)." class="ddia-figure ddia-figure--wide" width="1772" height="1065" />}}

### Property graph {#id56}

Trong mô hình *property graph* (còn gọi là *labeled property graph*, tức property graph có nhãn), mỗi đỉnh gồm:

* Một định danh duy nhất
* Một nhãn (chuỗi) mô tả loại đối tượng mà đỉnh biểu diễn
* Một tập các cạnh đi ra
* Một tập các cạnh đi vào
* Một tập thuộc tính (các cặp khóa-giá trị)

Mỗi cạnh gồm:

* Một định danh duy nhất
* Điểm bắt đầu của cạnh (*đỉnh đuôi*, tức *tail vertex*)
* Điểm kết thúc của cạnh (*đỉnh đầu*, tức *head vertex*)
* Một nhãn mô tả loại quan hệ giữa hai đỉnh
* Một tập thuộc tính (các cặp khóa-giá trị)

Có thể coi kho lưu trữ đồ thị gồm hai bảng quan hệ: một bảng lưu đỉnh, một bảng lưu cạnh, như trong {{< xref eg="3-3" page="/ch3" anchor="fig_graph_sql_schema" >}}Ví dụ 3-3{{< /xref >}} (schema này dùng kiểu dữ liệu `jsonb` của PostgreSQL để lưu thuộc tính của mỗi đỉnh hoặc mỗi cạnh). Mỗi cạnh lưu cả đỉnh đầu và đỉnh đuôi. Nếu muốn tìm mọi cạnh đi vào hay đi ra của một đỉnh, có thể truy vấn bảng `edges` theo `head_vertex` hoặc `tail_vertex` tương ứng.

{{< eg num="3-3" id="fig_graph_sql_schema" caption="Biểu diễn property graph bằng schema quan hệ" >}}
```sql
CREATE TABLE vertices (
    vertex_id integer PRIMARY KEY,
    label text,
    properties jsonb
);

CREATE TABLE edges (
    edge_id integer PRIMARY KEY,
    tail_vertex integer REFERENCES vertices (vertex_id),
    head_vertex integer REFERENCES vertices (vertex_id),
    label text,
    properties jsonb
);

CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```
{{< /eg >}}

Mô hình này có một số đặc điểm quan trọng:

1. Bất kỳ đỉnh nào cũng có thể được nối bằng cạnh với bất kỳ đỉnh nào khác. Không có schema nào hạn chế những thứ nào được phép liên kết với nhau và những thứ nào thì không.
2. Với một đỉnh bất kỳ, có thể tìm hiệu quả cả cạnh đi vào lẫn cạnh đi ra của nó, nhờ đó *duyệt* (*traverse*) đồ thị theo cả hai chiều — tức là đi tới và đi lui dọc một đường gồm chuỗi các đỉnh. (Đó chính là lý do {{< xref eg="3-3" page="/ch3" anchor="fig_graph_sql_schema" >}}Ví dụ 3-3{{< /xref >}} lập chỉ mục (*index*) cho cả hai cột `tail_vertex` và `head_vertex`.)
3. Dùng nhãn khác nhau cho các loại đỉnh và quan hệ khác nhau, ta có thể lưu nhiều loại thông tin trong một đồ thị mà vẫn giữ mô hình dữ liệu rõ ràng.

Bảng cạnh giống như bảng liên kết nhiều-nhiều (bảng join) mà ta đã thấy trong [“Quan hệ nhiều-một và nhiều-nhiều”](/ch3#sec_datamodels_many_to_many), chỉ khác là được tổng quát hóa để lưu nhiều loại quan hệ khác nhau trong cùng một bảng. Nhãn và thuộc tính cũng có thể được lập chỉ mục (*index*), giúp tìm hiệu quả các đỉnh hoặc cạnh có một thuộc tính nhất định.


> [!NOTE]
> Mô hình đồ thị có một hạn chế: một cạnh chỉ có thể nối hai đỉnh, trong khi bảng join của mô hình quan hệ có thể chứa nhiều tham chiếu khóa ngoại trong một hàng, nhờ đó biểu diễn được quan hệ bậc ba hoặc cao hơn. Trong đồ thị, có thể tạo thêm một đỉnh cho mỗi hàng của bảng join rồi nối nó với các đỉnh khác bằng cạnh, hoặc dùng *siêu đồ thị* (*hypergraph*) để biểu diễn loại quan hệ này.


Những đặc điểm này mang lại sự linh hoạt lớn cho việc mô hình hóa dữ liệu, như thể hiện trong {{< xref fig="3-6" page="/ch3" anchor="fig_datamodels_graph" >}}Hình 3-6{{< /xref >}}. Đồ thị này có một số thứ khó diễn đạt trong schema quan hệ truyền thống, chẳng hạn các quốc gia khác nhau có cấu trúc phân chia hành chính khác nhau (Pháp có *département* và *région*, Mỹ có *county* và *state*), những điều kỳ lạ của lịch sử như một nước nằm trong một nước khác (tạm bỏ qua mối quan hệ rắc rối giữa quốc gia có chủ quyền và các dân tộc), và dữ liệu có độ chi tiết khác nhau (nơi ở hiện tại của Lucy được ghi đến cấp thành phố, còn nơi sinh chỉ ghi đến cấp bang).

Có thể hình dung đồ thị này còn mở rộng được thêm nhiều sự kiện về Lucy, Alain hoặc những người khác. Chẳng hạn, có thể dùng nó để biểu diễn dị ứng thực phẩm: thêm một đỉnh cho mỗi chất gây dị ứng, dùng cạnh giữa người và chất gây dị ứng để biểu diễn dị ứng, rồi nối các chất gây dị ứng với một tập đỉnh cho biết thực phẩm nào chứa chất nào. Khi đó có thể viết một truy vấn tìm ra những thứ mỗi người có thể ăn an toàn. Đồ thị có ưu thế về khả năng tiến hóa: khi ứng dụng được bổ sung tính năng, có thể dễ dàng mở rộng đồ thị để thích ứng với những thay đổi của cấu trúc dữ liệu.

### Ngôn ngữ truy vấn Cypher {#id57}

*Cypher* là ngôn ngữ truy vấn cho property graph, ban đầu được tạo ra cho cơ sở dữ liệu đồ thị Neo4j, sau đó phát triển thành một chuẩn mở với tên *openCypher* [^38]. Ngoài Neo4j, Cypher còn được hỗ trợ bởi Memgraph, KùzuDB [^35], Amazon Neptune, Apache AGE (dữ liệu lưu trong PostgreSQL), v.v. Nó được đặt theo tên một nhân vật trong bộ phim *The Matrix*, và không liên quan gì đến mật mã (cipher) trong mật mã học [^39].

{{< xref eg="3-4" page="/ch3" anchor="fig_cypher_create" >}}Ví dụ 3-4{{< /xref >}} cho thấy truy vấn Cypher chèn phần bên trái của {{< xref fig="3-6" page="/ch3" anchor="fig_datamodels_graph" >}}Hình 3-6{{< /xref >}} vào cơ sở dữ liệu đồ thị; phần còn lại của đồ thị có thể được thêm theo cách tương tự. Mỗi đỉnh được đặt một tên ký hiệu như `usa` hoặc `idaho`. Tên này không được lưu vào cơ sở dữ liệu, mà chỉ dùng bên trong truy vấn để tạo cạnh giữa các đỉnh. Ký pháp mũi tên `(idaho) -[:WITHIN]-> (usa)` tạo một cạnh có nhãn `WITHIN`, với `idaho` là nút đuôi và `usa` là nút đầu.

{{< eg num="3-4" id="fig_cypher_create" caption="Một phần dữ liệu của Hình 3-6, biểu diễn bằng truy vấn Cypher" >}}
```
CREATE
    (namerica :Location {name:'North America', type:'continent'}),
    (usa :Location {name:'United States', type:'country' }),
    (idaho :Location {name:'Idaho', type:'state' }),
    (lucy :Person {name:'Lucy' }),
    (idaho) -[:WITHIN ]-> (usa) -[:WITHIN]-> (namerica),
    (lucy) -[:BORN_IN]-> (idaho)
```
{{< /eg >}}

Sau khi thêm mọi đỉnh và cạnh của {{< xref fig="3-6" page="/ch3" anchor="fig_datamodels_graph" >}}Hình 3-6{{< /xref >}} vào cơ sở dữ liệu, ta có thể đặt ra những câu hỏi thú vị. Chẳng hạn: *tìm tên của tất cả những người đã di cư từ Hoa Kỳ sang châu Âu*. Chính xác hơn, ta muốn tìm các đỉnh vừa có một cạnh `BORN_IN` trỏ tới một địa điểm nằm trong Hoa Kỳ, vừa có một cạnh `LIVES_IN` trỏ tới một địa điểm nằm trong châu Âu, rồi trả về thuộc tính `name` của các đỉnh đó.

{{< xref eg="3-5" page="/ch3" anchor="fig_cypher_query" >}}Ví dụ 3-5{{< /xref >}} cho thấy cách diễn đạt truy vấn này bằng Cypher. Mệnh đề `MATCH` dùng cùng ký pháp mũi tên để tìm mẫu trong đồ thị: `(person) -[:BORN_IN]-> ()` khớp với hai đỉnh bất kỳ được nối bằng một cạnh `BORN_IN`; đỉnh đuôi của cạnh này được gán cho biến `person`, còn đỉnh đầu thì không đặt tên.

{{< eg num="3-5" id="fig_cypher_query" caption="Truy vấn Cypher tìm những người di cư từ Hoa Kỳ sang châu Âu" >}}
```
MATCH
    (person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (:Location {name:'United States'}),
    (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (:Location {name:'Europe'})
RETURN person.name
```
{{< /eg >}}

Truy vấn này có thể đọc như sau:

> Tìm mọi đỉnh (gọi là `person`) thỏa mãn *cả hai* điều kiện sau:
>
> 1. Đỉnh `person` có một cạnh `BORN_IN` đi ra, trỏ tới một đỉnh nào đó. Từ đó, đi theo một chuỗi các cạnh `WITHIN` đi ra thì cuối cùng đến được một đỉnh thuộc loại `Location` có thuộc tính `name` là `"United States"`.
> 2. Chính đỉnh `person` đó cũng có một cạnh `LIVES_IN` đi ra. Đi theo cạnh này, rồi đi theo một chuỗi các cạnh `WITHIN` đi ra, cuối cùng đến được một đỉnh thuộc loại `Location` có thuộc tính `name` là `"Europe"`.
>
> Với mỗi đỉnh `person` như vậy, trả về thuộc tính `name` của nó.

Có nhiều cách khả thi để thực thi truy vấn này. Mô tả ở trên gợi ý rằng ta có thể quét toàn bộ những người trong cơ sở dữ liệu, kiểm tra từng người về nơi sinh và nơi cư trú, và chỉ trả về những người phù hợp.

Tương đương, cũng có thể bắt đầu từ hai đỉnh `Location` và tìm ngược lại. Nếu thuộc tính `name` được lập chỉ mục (*index*), có thể tìm hiệu quả hai đỉnh đại diện cho Hoa Kỳ và châu Âu. Sau đó đi theo mọi cạnh `WITHIN` đi vào để tìm tất cả địa điểm (bang, vùng, thành phố, v.v.) nằm trong Hoa Kỳ và châu Âu. Cuối cùng, đi theo các cạnh `BORN_IN` hoặc `LIVES_IN` đi vào của các đỉnh địa điểm đó để tìm những người tương ứng.

### Truy vấn đồ thị trong SQL {#id58}

{{< xref eg="3-3" page="/ch3" anchor="fig_graph_sql_schema" >}}Ví dụ 3-3{{< /xref >}} cho thấy có thể biểu diễn dữ liệu đồ thị trong cơ sở dữ liệu quan hệ. Nhưng nếu dữ liệu đồ thị được lưu theo cấu trúc quan hệ, ta có thể truy vấn nó bằng SQL không?

Câu trả lời là có, nhưng khá khó khăn. Mỗi lần đi qua một cạnh trong truy vấn đồ thị thực chất tương đương với một lần join với bảng `edges`. Trong cơ sở dữ liệu quan hệ, thường biết trước truy vấn cần những phép join nào, còn trong truy vấn đồ thị, có thể phải đi qua một số lượng cạnh không cố định trước khi tìm thấy đỉnh cần tìm, nghĩa là không thể biết trước số phép join.

Trong ví dụ của ta, mẫu `() -[:WITHIN*0..]-> ()` trong truy vấn Cypher chính là trường hợp như vậy. Cạnh `LIVES_IN` của một người có thể trỏ tới địa điểm ở bất kỳ cấp nào: đường phố, thành phố, quận, vùng, bang, v.v. Một thành phố có thể nằm `WITHIN` một vùng, vùng nằm `WITHIN` một bang, bang lại nằm `WITHIN` một quốc gia. Cạnh `LIVES_IN` có thể trỏ thẳng tới địa điểm cần tìm, hoặc còn cách nó vài tầng trong hệ thống phân cấp địa điểm.

Cypher diễn đạt điều này rất gọn bằng `:WITHIN*0..`: “đi theo cạnh `WITHIN` không hoặc nhiều lần”. Nó giống toán tử `*` trong biểu thức chính quy.

Từ SQL:1999, có thể biểu diễn các đường duyệt có độ dài thay đổi trong truy vấn bằng cái gọi là *biểu thức bảng chung đệ quy* (*recursive common table expression*, cú pháp `WITH RECURSIVE`). {{< xref eg="3-6" page="/ch3" anchor="fig_graph_sql_query" >}}Ví dụ 3-6{{< /xref >}} dùng kỹ thuật này để viết lại cùng truy vấn đó trong SQL — tìm tên những người di cư từ Hoa Kỳ sang châu Âu. Có điều, so với Cypher thì cú pháp của nó rất vụng về.

{{< eg num="3-6" id="fig_graph_sql_query" caption="Cùng truy vấn như Ví dụ 3-5, viết bằng SQL với biểu thức bảng chung đệ quy" >}}
```sql
WITH RECURSIVE

    -- in_usa là tập ID đỉnh của mọi địa điểm nằm trong Hoa Kỳ
    in_usa(vertex_id) AS (
        SELECT vertex_id FROM vertices
            WHERE label = 'Location' AND properties->>'name' = 'United States' ❶ 
      UNION
        SELECT edges.tail_vertex FROM edges ❷
            JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
            WHERE edges.label = 'within'
    ),
    
    -- in_europe là tập ID đỉnh của mọi địa điểm nằm trong châu Âu
    in_europe(vertex_id) AS (
        SELECT vertex_id FROM vertices
            WHERE label = 'location' AND properties->>'name' = 'Europe' ❸
      UNION
        SELECT edges.tail_vertex FROM edges
            JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
            WHERE edges.label = 'within'
    ),
    
    -- born_in_usa là tập ID đỉnh của mọi người sinh ra ở Hoa Kỳ
    born_in_usa(vertex_id) AS ( ❹
        SELECT edges.tail_vertex FROM edges
            JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
            WHERE edges.label = 'born_in'
    ),
    
    -- lives_in_europe là tập ID đỉnh của mọi người đang sống ở châu Âu
    lives_in_europe(vertex_id) AS ( ❺
        SELECT edges.tail_vertex FROM edges
            JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
            WHERE edges.label = 'lives_in'
    )
    
    SELECT vertices.properties->>'name'
    FROM vertices
    -- join để tìm những người vừa sinh ra ở Hoa Kỳ *và* đang sống ở châu Âu
    JOIN born_in_usa ON vertices.vertex_id = born_in_usa.vertex_id ❻
    JOIN lives_in_europe ON vertices.vertex_id = lives_in_europe.vertex_id;
```
{{< /eg >}}

❶: Trước hết tìm đỉnh có thuộc tính `name` là `"United States"`, và coi nó là phần tử đầu tiên của tập đỉnh `in_usa`.

❷: Từ các đỉnh trong tập `in_usa`, đi ngược theo mọi cạnh `within` đi vào, thêm các đỉnh đến được vào cùng tập đó, cho đến khi mọi cạnh `within` đi vào đều đã được duyệt.

❸: Làm tương tự bắt đầu từ đỉnh có thuộc tính `name` là `"Europe"`, để xây dựng tập đỉnh `in_europe`.

❹: Với mỗi đỉnh trong tập `in_usa`, đi theo các cạnh `born_in` đi vào để tìm những người sinh ra ở một nơi nào đó trong Hoa Kỳ.

❺: Tương tự, với mỗi đỉnh trong tập `in_europe`, đi theo các cạnh `lives_in` đi vào để tìm những người đang sống ở châu Âu.

❻: Cuối cùng, dùng join để lấy giao của hai tập “người sinh ra ở Hoa Kỳ” và “người sống ở châu Âu”.

Cùng một truy vấn, Cypher chỉ cần 4 dòng còn SQL phải viết tới 31 dòng, điều này cho thấy việc chọn đúng mô hình dữ liệu và ngôn ngữ truy vấn tạo ra khác biệt lớn đến mức nào. Và đây mới chỉ là khởi đầu, còn nhiều chi tiết khác cần cân nhắc, chẳng hạn cách xử lý chu trình, và chọn duyệt theo chiều rộng hay chiều sâu [^40].

Oracle cung cấp một bộ mở rộng SQL khác cho truy vấn đệ quy, gọi là *truy vấn phân cấp* (*hierarchical query*) [^41].

Tuy nhiên, tình hình có thể đang cải thiện: tại thời điểm viết sách này, đã có kế hoạch đưa một ngôn ngữ truy vấn đồ thị tên là GQL vào chuẩn SQL [^42] [^43]. Cú pháp của nó mượn từ Cypher, GSQL [^44] và PGQL [^45].

### Triple store và SPARQL {#id59}

Mô hình triple store về cơ bản giống mô hình property graph, chỉ là dùng từ vựng khác để diễn tả cùng một ý tưởng. Dù vậy nó vẫn đáng được bàn riêng, vì triple store có nhiều công cụ và ngôn ngữ sẵn có, có thể là phần bổ sung quý giá khi xây dựng ứng dụng.

Trong triple store, mọi thông tin được lưu dưới dạng các phát biểu ba phần rất đơn giản: (*chủ ngữ*, *subject*; *vị ngữ*, *predicate*; *tân ngữ*, *object*). Ví dụ, trong bộ ba (*Jim*, *thích*, *chuối*), *Jim* là chủ ngữ, *thích* là vị ngữ (động từ) và *chuối* là tân ngữ.

Chủ ngữ của một bộ ba tương đương một đỉnh trong đồ thị, còn tân ngữ là một trong hai loại sau:

1. Giá trị thuộc kiểu dữ liệu nguyên thủy như chuỗi hay số. Khi đó, vị ngữ và tân ngữ của bộ ba tương đương khóa và giá trị của một thuộc tính trên đỉnh chủ ngữ. Ví dụ, theo ví dụ ở {{< xref fig="3-6" page="/ch3" anchor="fig_datamodels_graph" >}}Hình 3-6{{< /xref >}}, bộ ba (*lucy*, *birthYear*, *1989*) tương đương đỉnh `lucy` có thuộc tính `{"birthYear": 1989}`.
2. Một đỉnh khác trong đồ thị. Khi đó, vị ngữ tương đương cạnh của đồ thị, chủ ngữ là đỉnh đuôi và tân ngữ là đỉnh đầu. Ví dụ, trong (*lucy*, *marriedTo*, *alain*), *lucy* và *alain* đều là đỉnh, còn vị ngữ *marriedTo* là nhãn của cạnh nối hai đỉnh này.

> [!NOTE]
> Nói chặt chẽ, các cơ sở dữ liệu cung cấp mô hình dữ liệu kiểu bộ ba thường còn lưu thêm một số siêu dữ liệu cho mỗi bộ. Ví dụ, AWS Neptune dùng bộ bốn (4-tuple), tức thêm một ID đồ thị vào mỗi bộ ba [^46]; Datomic dùng bộ năm, thêm vào mỗi bộ ba một ID giao dịch và một giá trị boolean biểu thị việc xóa [^47]. Các cơ sở dữ liệu này vẫn giữ cấu trúc cơ bản *chủ ngữ — vị ngữ — tân ngữ* như trên, nên sách này vẫn gọi chung chúng là triple store.

{{< xref eg="3-7" page="/ch3" anchor="fig_graph_n3_triples" >}}Ví dụ 3-7{{< /xref >}} viết lại cùng dữ liệu trong {{< xref eg="3-4" page="/ch3" anchor="fig_cypher_create" >}}Ví dụ 3-4{{< /xref >}} dưới dạng bộ ba, theo định dạng gọi là *Turtle*, một tập con của *Notation3* (*N3*) [^48].

{{< eg num="3-7" id="fig_graph_n3_triples" caption="Một phần dữ liệu của Hình 3-6, biểu diễn dưới dạng bộ ba Turtle" >}}
```
@prefix : <urn:example:>.
_:lucy a :Person.
_:lucy :name "Lucy".
_:lucy :bornIn _:idaho.
_:idaho a :Location.
_:idaho :name "Idaho".
_:idaho :type "state".
_:idaho :within _:usa.
_:usa a :Location.
_:usa :name "United States".
_:usa :type "country".
_:usa :within _:namerica.
_:namerica a :Location.
_:namerica :name "North America".
_:namerica :type "continent".
```
{{< /eg >}}

Trong ví dụ này, các đỉnh của đồ thị được viết là `_:someName`. Tên này không có ý nghĩa bên ngoài tệp hiện tại. Nó chỉ cần thiết để phân biệt những bộ ba nào tham chiếu đến cùng một đỉnh. Khi vị ngữ biểu thị cạnh, tân ngữ là một đỉnh khác, như `_:idaho :within _:usa`. Khi vị ngữ biểu thị thuộc tính, tân ngữ là một chuỗi literal, như `_:usa :name "United States"`.

Lặp đi lặp lại cùng một chủ ngữ khá dài dòng, may là có thể dùng dấu chấm phẩy để nêu liên tiếp nhiều điều về cùng một chủ ngữ. Điều này làm định dạng Turtle khá rõ ràng và dễ đọc, như trong {{< xref eg="3-8" page="/ch3" anchor="fig_graph_n3_shorthand" >}}Ví dụ 3-8{{< /xref >}}.

{{< eg num="3-8" id="fig_graph_n3_shorthand" caption="Cách viết gọn hơn cho dữ liệu trong Ví dụ 3-7" >}}
```
@prefix : <urn:example:>.
_:lucy a :Person; :name "Lucy"; :bornIn _:idaho.
_:idaho a :Location; :name "Idaho"; :type "state"; :within _:usa.
_:usa a :Location; :name "United States"; :type "country"; :within _:namerica.
_:namerica a :Location; :name "North America"; :type "continent".
```
{{< /eg >}}

> [!TIP] Semantic Web
>
> Một phần nghiên cứu và phát triển về triple store bắt nguồn từ sự thúc đẩy của *Semantic Web*. Nỗ lực này bắt đầu từ đầu những năm 2000, với mong muốn dữ liệu không chỉ được công bố dưới dạng trang web cho con người đọc mà còn dưới dạng chuẩn hóa, máy đọc được, nhờ đó thúc đẩy trao đổi dữ liệu trên toàn Internet. Semantic Web như ban đầu hình dung đã không thành công [^49] [^50], nhưng dự án vẫn để lại một số công nghệ cụ thể: các chuẩn *linked data* như JSON-LD [^51], các *ontology* dùng trong y sinh [^52], giao thức Open Graph của Facebook [^53] (dùng để hiển thị bản xem trước liên kết [^54]), các knowledge graph như Wikidata, và bộ từ vựng chuẩn cho dữ liệu có cấu trúc do [`schema.org`](https://schema.org/) duy trì.
>
> Triple store cũng là một công nghệ đã vượt ra khỏi bối cảnh gốc của Semantic Web và tìm được chỗ dùng ở nơi khác: ngay cả khi bạn không hứng thú gì với Semantic Web, bộ ba vẫn có thể là một mô hình dữ liệu nội bộ tốt cho ứng dụng.

#### Mô hình dữ liệu RDF {#the-rdf-data-model}

Ngôn ngữ Turtle dùng trong {{< xref eg="3-8" page="/ch3" anchor="fig_graph_n3_shorthand" >}}Ví dụ 3-8{{< /xref >}} thực ra là một cách mã hóa dữ liệu *Resource Description Framework* (RDF) [^55]; RDF là mô hình dữ liệu được thiết kế riêng cho *Semantic Web*. Dữ liệu RDF cũng có thể dùng các cách mã hóa khác, chẳng hạn XML dài dòng hơn, như trong {{< xref eg="3-9" page="/ch3" anchor="fig_graph_rdf_xml" >}}Ví dụ 3-9{{< /xref >}}. Các công cụ như Apache Jena có thể tự động chuyển đổi giữa các cách mã hóa RDF.

{{< eg num="3-9" id="fig_graph_rdf_xml" caption="Biểu diễn dữ liệu của Ví dụ 3-8 bằng cú pháp RDF/XML" >}}
```xml
<rdf:RDF xmlns="urn:example:"
         xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">

    <Location rdf:nodeID="idaho">
        <name>Idaho</name>
        <type>state</type>
        <within>
            <Location rdf:nodeID="usa">
                <name>United States</name>
                <type>country</type>
                <within>
                    <Location rdf:nodeID="namerica">
                        <name>North America</name>
                        <type>continent</type>
                    </Location>
                </within>
            </Location>
        </within>
    </Location>

    <Person rdf:nodeID="lucy">
        <name>Lucy</name>
        <bornIn rdf:nodeID="idaho"/>
    </Person>
</rdf:RDF>
```
{{< /eg >}}

RDF có vài điểm kỳ lạ vì nó được thiết kế cho việc trao đổi dữ liệu trên phạm vi Internet. Chủ ngữ, vị ngữ và tân ngữ của một bộ ba thường đều là URI. Ví dụ, vị ngữ có thể được viết là `<http://my-company.com/namespace#within>` hoặc `<http://my-company.com/namespace#lives_in>` thay vì chỉ `WITHIN` hay `LIVES_IN`. Thiết kế này nhằm cho phép gộp dữ liệu từ các nguồn khác nhau: ngay cả khi người khác gán cho `within` hay `lives_in` một ý nghĩa khác, hai bên cũng không xung đột, vì vị ngữ của họ thực ra là `<http://other.org/foo#within>` và `<http://other.org/foo#lives_in>`.

Theo quan điểm của RDF, URL `<http://my-company.com/namespace>` không nhất thiết phải phân giải ra nội dung nào. Nó chỉ là một namespace. Để tránh nhầm với URL `http://`, các ví dụ trong mục này dùng URI không phân giải được như `urn:example:within`. May là chỉ cần khai báo tiền tố một lần ở đầu tệp, về sau không phải bận tâm nữa.

#### Ngôn ngữ truy vấn SPARQL {#the-sparql-query-language}

*SPARQL* là ngôn ngữ truy vấn triple store dành cho mô hình dữ liệu RDF [^56]. (Đây là viết tắt của *SPARQL Protocol and RDF Query Language*, đọc là “sparkle”.) SPARQL ra đời trước Cypher. Việc khớp mẫu của Cypher mượn từ SPARQL nên hai ngôn ngữ trông rất giống nhau.

Truy vấn tìm người di cư từ Hoa Kỳ sang châu Âu ở trên, khi viết bằng SPARQL cũng ngắn gọn như bằng Cypher (xem {{< xref eg="3-10" page="/ch3" anchor="fig_sparql_query" >}}Ví dụ 3-10{{< /xref >}}).

{{< eg num="3-10" id="fig_sparql_query" caption="Cùng truy vấn như Ví dụ 3-5, biểu diễn bằng SPARQL" >}}
```
PREFIX : <urn:example:>

SELECT ?personName WHERE {
 ?person :name ?personName.
 ?person :bornIn / :within* / :name "United States".
 ?person :livesIn / :within* / :name "Europe".
}
```
{{< /eg >}}

Hai cấu trúc rất giống nhau. Hai biểu thức dưới đây tương đương (biến trong SPARQL bắt đầu bằng dấu hỏi):

```
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (location) # Cypher

?person :bornIn / :within* ?location. # SPARQL
```

Vì RDF không phân biệt thuộc tính với cạnh mà coi cả hai là vị ngữ, nên việc khớp thuộc tính cũng dùng được cùng một cú pháp. Trong biểu thức dưới đây, biến `usa` sẽ được gán cho bất kỳ đỉnh nào có thuộc tính `name` là chuỗi `"United States"`:

```
(usa {name:'United States'}) # Cypher

?usa :name "United States". # SPARQL
```

Amazon Neptune, AllegroGraph, Blazegraph, OpenLink Virtuoso, Apache Jena và nhiều triple store khác đều hỗ trợ SPARQL [^36].

### Datalog: truy vấn quan hệ đệ quy {#id62}

Datalog là ngôn ngữ cổ hơn SPARQL và Cypher, bắt nguồn từ nghiên cứu học thuật thập niên 1980 [^57] [^58] [^59]. Nó ít được kỹ sư phần mềm biết đến, và các cơ sở dữ liệu phổ biến cũng hiếm khi hỗ trợ, nhưng nó có sức biểu đạt rất mạnh, đặc biệt giỏi các truy vấn phức tạp, và xứng đáng được chú ý hơn. Một số cơ sở dữ liệu ngách như Datomic, LogicBlox, CozoDB và LIquid của LinkedIn [^60] đều dùng Datalog làm ngôn ngữ truy vấn.

Datalog thực chất dựa trên mô hình dữ liệu quan hệ chứ không phải mô hình đồ thị. Sở dĩ đặt nó vào mục cơ sở dữ liệu đồ thị là vì Datalog đặc biệt giỏi truy vấn đệ quy trên đồ thị.

Cơ sở dữ liệu Datalog gồm các *dữ kiện* (*fact*), mỗi dữ kiện tương ứng một hàng trong bảng quan hệ. Giả sử có bảng *location* lưu các địa điểm với ba cột *ID*, *name* và *type*. Dữ kiện “Hoa Kỳ là một quốc gia” có thể viết là `location(2, "United States", "country")`, trong đó `2` là ID của Hoa Kỳ. Nói chung, `table(val1, val2, …​)` nghĩa là trong `table` có một hàng mà cột thứ nhất là `val1`, cột thứ hai là `val2`, và cứ thế.

{{< xref eg="3-11" page="/ch3" anchor="fig_datalog_triples" >}}Ví dụ 3-11{{< /xref >}} cho thấy cách viết dữ liệu ở phía trái {{< xref fig="3-6" page="/ch3" anchor="fig_datamodels_graph" >}}Hình 3-6{{< /xref >}} bằng Datalog. Các cạnh của đồ thị (`within`, `born_in` và `lives_in`) được biểu diễn thành bảng nối hai cột. Ví dụ, ID của Lucy là 100 và ID của bang Idaho là 3, nên quan hệ “Lucy sinh ra ở Idaho” được biểu diễn là `born_in(100, 3)`.

{{< eg num="3-11" id="fig_datalog_triples" caption="Một tập con dữ liệu của Hình 3-6, biểu diễn dưới dạng dữ kiện Datalog" >}}
```
location(1, "North America", "continent").
location(2, "United States", "country").
location(3, "Idaho", "state").

within(2, 1). /* Hoa Kỳ nằm trong Bắc Mỹ */
within(3, 2). /* Idaho nằm trong Hoa Kỳ */

person(100, "Lucy").
born_in(100, 3). /* Lucy sinh ra ở Idaho */
```
{{< /eg >}}

Khi đã định nghĩa xong dữ liệu, ta có thể viết truy vấn giống như trước, như trong {{< xref eg="3-12" page="/ch3" anchor="fig_datalog_query" >}}Ví dụ 3-12{{< /xref >}}. Nó trông khá khác truy vấn tương đương trong Cypher hay SPARQL, nhưng đừng vì thế mà chùn bước. Datalog là một tập con của Prolog. Nếu từng học khoa học máy tính, có thể bạn đã gặp ngôn ngữ lập trình này.

{{< eg num="3-12" id="fig_datalog_query" caption="Cùng truy vấn như Ví dụ 3-5, biểu diễn bằng Datalog" >}}
```sql
within_recursive(LocID, PlaceName) :- location(LocID, PlaceName, _). /* quy tắc 1 */

within_recursive(LocID, PlaceName) :- within(LocID, ViaID), /* quy tắc 2 */
 within_recursive(ViaID, PlaceName).

migrated(PName, BornIn, LivingIn) :- person(PersonID, PName), /* quy tắc 3 */
 born_in(PersonID, BornID),
 within_recursive(BornID, BornIn),
 lives_in(PersonID, LivingID),
 within_recursive(LivingID, LivingIn).

us_to_europe(Person) :- migrated(Person, "United States", "Europe"). /* quy tắc 4 */
/* us_to_europe chứa hàng "Lucy". */
```
{{< /eg >}}

Cypher và SPARQL lao vào `SELECT` ngay, còn Datalog mỗi lần chỉ tiến một bước nhỏ. Ta định nghĩa các *quy tắc* (*rule*) để từ các dữ kiện nền suy ra những bảng ảo mới. Các bảng suy diễn này giống view (ảo) của SQL: chúng không được lưu trong cơ sở dữ liệu, nhưng vẫn có thể được truy vấn như các bảng chứa dữ kiện.

{{< xref eg="3-12" page="/ch3" anchor="fig_datalog_query" >}}Ví dụ 3-12{{< /xref >}} định nghĩa ba bảng suy diễn: `within_recursive`, `migrated` và `us_to_europe`. Phần đứng trước ký hiệu `:-` trong mỗi quy tắc định nghĩa tên và các cột của bảng ảo. Ví dụ, `migrated(PName, BornIn, LivingIn)` là bảng ba cột, lần lượt chứa tên người, tên nơi sinh và tên nơi cư trú.

Nội dung của bảng ảo được định nghĩa bởi phần đứng sau ký hiệu `:-`, phần này cố tìm trong các bảng những hàng khớp một mẫu nhất định. Ví dụ, `person(PersonID, PName)` có thể khớp hàng `person(100, "Lucy")`, khi đó biến `PersonID` được gán là `100` và `PName` là `"Lucy"`. Chỉ cần hệ thống tìm được khớp cho *tất cả* các mẫu ở vế phải `:-`, quy tắc có thể được áp dụng. Hiệu quả của việc áp dụng quy tắc giống như thêm vào cơ sở dữ liệu nội dung vế trái `:-`, với các biến được thay bằng giá trị chúng khớp.

Do đó, có thể áp dụng các quy tắc như sau (như trong {{< xref fig="3-7" page="/ch3" anchor="fig_datalog_naive" >}}Hình 3-7{{< /xref >}}):

1. Cơ sở dữ liệu có `location(1, "North America", "continent")`, nên quy tắc 1 áp dụng được, sinh ra `within_recursive(1, "North America")`.
2. Cơ sở dữ liệu có `within(2, 1)`, và bước trước đã sinh ra `within_recursive(1, "North America")`, nên quy tắc 2 áp dụng được, sinh ra `within_recursive(2, "North America")`.
3. Cơ sở dữ liệu có `within(3, 2)`, và bước trước đã sinh ra `within_recursive(2, "North America")`, nên áp dụng quy tắc 2 lần nữa, sinh ra `within_recursive(3, "North America")`.

Áp dụng lặp lại quy tắc 1 và quy tắc 2, bảng ảo `within_recursive` cho ta biết những địa điểm nào trong cơ sở dữ liệu nằm trong Bắc Mỹ (hoặc bất kỳ địa điểm nào khác).

{{< fig num="3-7" id="fig_datalog_naive" src="/fig/ddia_0307.png" caption="Dùng các quy tắc Datalog trong Ví dụ 3-12 để xác định Idaho nằm trong Bắc Mỹ." link="#fig_datalog_query" class="ddia-figure ddia-figure--panorama" width="1772" height="585" />}}



Tiếp theo, quy tắc 3 tìm ra những người sinh ở `BornIn` và hiện sống ở `LivingIn`. Quy tắc 4 gọi quy tắc 3 với `BornIn = 'United States'` và `LivingIn = 'Europe'`, chỉ trả về tên những người thỏa điều kiện. Cuối cùng truy vấn bảng ảo `us_to_europe`, hệ thống Datalog sẽ cho ra cùng đáp án như các truy vấn Cypher và SPARQL trước đó.

Datalog đòi hỏi một cách tư duy khác với các ngôn ngữ truy vấn khác trong chương này. Nó cho phép xây truy vấn phức tạp từng quy tắc một, để quy tắc này tham chiếu quy tắc khác, giống như tách mã thành các hàm gọi lẫn nhau. Hàm có thể đệ quy, quy tắc Datalog cũng có thể tự gọi chính nó. Quy tắc 2 trong {{< xref eg="3-12" page="/ch3" anchor="fig_datalog_query" >}}Ví dụ 3-12{{< /xref >}} chính là như vậy, nhờ đó hiện thực hóa việc duyệt đồ thị trong truy vấn Datalog.

### GraphQL {#id63}

GraphQL cũng là một ngôn ngữ truy vấn, nhưng được thiết kế hạn chế hơn các ngôn ngữ khác trong chương này. Mục đích của GraphQL là để phần mềm client chạy trên thiết bị của người dùng (chẳng hạn ứng dụng di động hoặc frontend của ứng dụng web JavaScript) yêu cầu một tài liệu JSON có cấu trúc cụ thể, chứa đúng các trường cần để render giao diện người dùng. Nhờ giao diện GraphQL, nhà phát triển có thể nhanh chóng sửa truy vấn trong mã client mà không cần đổi API phía server.

Tính linh hoạt của GraphQL không phải không có giá. Các tổ chức dùng GraphQL thường cần một bộ công cụ để chuyển truy vấn GraphQL thành yêu cầu tới các dịch vụ nội bộ, vốn thường dùng REST hoặc gRPC (xem [Chương 5](/ch5#ch_encoding)). Ngoài ra còn phải xử lý các vấn đề như ủy quyền, giới hạn tốc độ và hiệu năng [^61]. Truy vấn GraphQL đến từ nguồn không đáng tin, nên bản thân ngôn ngữ truy vấn cũng bị hạn chế chặt: nó không cho phép bất kỳ thao tác nào có thể tốn kém khi thực thi, nếu không người dùng có thể gửi hàng loạt truy vấn đắt đỏ để tấn công từ chối dịch vụ vào server. Cụ thể, GraphQL không cho phép truy vấn đệ quy (Cypher, SPARQL, SQL và Datalog đều cho phép), cũng không cho phép đưa ra điều kiện tìm kiếm tùy ý như “tìm người sinh ở Hoa Kỳ và hiện sống ở châu Âu”, trừ khi chủ sở hữu dịch vụ cố ý cung cấp chức năng tìm kiếm đó.

Dù vậy, GraphQL vẫn rất hữu ích. {{< xref eg="3-13" page="/ch3" anchor="fig_graphql_query" >}}Ví dụ 3-13{{< /xref >}} cho thấy cách dùng nó để hiện thực một ứng dụng chat nhóm như Discord hay Slack. Truy vấn yêu cầu tất cả kênh mà người dùng có quyền truy cập, và lấy về tên của từng kênh cùng 50 tin nhắn gần nhất. Với mỗi tin nhắn, nó yêu cầu dấu thời gian, nội dung, cùng tên và URL ảnh đại diện của người gửi. Nếu một tin nhắn là câu trả lời cho tin nhắn khác, truy vấn còn yêu cầu nội dung và tên người gửi của tin nhắn gốc. Giao diện có thể hiển thị những thông tin này bằng cỡ chữ nhỏ hơn phía trên câu trả lời, làm ngữ cảnh.

{{< eg num="3-13" id="fig_graphql_query" caption="Ví dụ truy vấn GraphQL cho ứng dụng chat nhóm" >}}
```
query ChatApp {
    channels {
        name
        recentMessages(latest: 50) {
            timestamp
            content
        sender {
            fullName
            imageUrl
        }
    replyTo {
        content
        sender {
            fullName
        }
    }
    }
    }
}
```
{{< /eg >}}

{{< xref eg="3-14" page="/ch3" anchor="fig_graphql_response" >}}Ví dụ 3-14{{< /xref >}} đưa ra một phản hồi có thể có cho truy vấn trong {{< xref eg="3-13" page="/ch3" anchor="fig_graphql_query" >}}Ví dụ 3-13{{< /xref >}}. Phản hồi là một tài liệu JSON có cấu trúc tương ứng với truy vấn: yêu cầu thuộc tính nào thì trả về đúng thuộc tính đó, không hơn không kém. Nhờ vậy server không cần biết trước client cần gì để render giao diện; client chỉ cần nêu thẳng thứ mình cần. Ví dụ, truy vấn này không yêu cầu URL ảnh đại diện của người gửi tin nhắn `replyTo`. Nếu sau này giao diện cần hiển thị ảnh đó, client chỉ cần thêm thuộc tính `imageUrl` vào truy vấn, không phải sửa server.

{{< eg num="3-14" id="fig_graphql_response" caption="Một phản hồi có thể có cho truy vấn trong Ví dụ 3-13" >}}
```json
{
"data": {
    "channels": [
        {
        "name": "#general",
        "recentMessages": [
        {
        "timestamp": 1693143014,
        "content": "Hey! How are y'all doing?",
        "sender": {"fullName": "Aaliyah", "imageUrl": "https://..."},
        "replyTo": null
        },
        {
            "timestamp": 1693143024,
            "content": "Great! And you?",
            "sender": {"fullName": "Caleb", "imageUrl": "https://..."},
            "replyTo": {
            "content": "Hey! How are y'all doing?",
            "sender": {"fullName": "Aaliyah"}
        }
},
...
```
{{< /eg >}}

{{< xref eg="3-14" page="/ch3" anchor="fig_graphql_response" >}}Ví dụ 3-14{{< /xref >}} nhúng tên và URL ảnh đại diện của người gửi thẳng vào đối tượng tin nhắn. Nếu cùng một người dùng gửi nhiều tin nhắn, thông tin này bị lặp lại trong từng tin nhắn. Về nguyên tắc có thể giảm sự lặp lại này, nhưng GraphQL chọn chấp nhận phản hồi lớn hơn để đổi lấy việc render giao diện từ dữ liệu đơn giản hơn.

Trường `replyTo` cũng vậy: tin nhắn thứ hai trong {{< xref eg="3-14" page="/ch3" anchor="fig_graphql_response" >}}Ví dụ 3-14{{< /xref >}} trả lời tin nhắn thứ nhất, nên nội dung tin nhắn thứ nhất (“Hey!…”) và người gửi Aaliyah lại được lặp lần nữa dưới `replyTo`. Cũng có thể chỉ trả về ID của tin nhắn được trả lời, nhưng nếu ID đó không nằm trong 50 tin nhắn gần nhất của lần trả về này, client sẽ phải gửi thêm một yêu cầu tới server. Sao chép (*replication*) thẳng nội dung thì xử lý đơn giản hơn nhiều.

Cơ sở dữ liệu phía server có thể lưu dữ liệu ở dạng chuẩn hóa hơn và thực hiện các phép nối cần thiết khi xử lý truy vấn. Ví dụ, server có thể lưu nội dung tin nhắn cùng ID người dùng của người gửi và ID của tin nhắn được trả lời. Khi nhận truy vấn như trên, nó mới phân giải các ID đó để tìm bản ghi chúng tham chiếu. Tuy nhiên, client chỉ có thể yêu cầu server thực hiện những phép nối được mở rõ ràng trong schema GraphQL.

Dù phản hồi GraphQL trông rất giống kết quả do cơ sở dữ liệu tài liệu trả về, và tên gọi lại có chữ “graph”, thực ra nó có thể được xây trên bất kỳ cơ sở dữ liệu nào, dù là cơ sở dữ liệu quan hệ, tài liệu hay đồ thị.


## Event sourcing và CQRS {#sec_datamodels_events}

Trong tất cả các mô hình dữ liệu đã bàn đến giờ, dữ liệu được truy vấn đúng ở dạng lúc nó được ghi — dù đó là tài liệu JSON, hàng trong bảng, hay đỉnh và cạnh trong đồ thị. Tuy nhiên, trong các ứng dụng phức tạp, đôi khi rất khó tìm được một cách biểu diễn dữ liệu đáp ứng được mọi nhu cầu truy vấn và trình bày dữ liệu. Khi đó, ta có thể ghi dữ liệu ở một dạng, rồi từ đó dẫn xuất ra nhiều cách biểu diễn, mỗi cách tối ưu cho một kiểu đọc khác nhau.

Ta đã gặp ý tưởng này ở [“Hệ thống ghi nhận gốc và dữ liệu dẫn xuất”](/ch1#sec_introduction_derived), nơi ETL (xem [“Kho dữ liệu”](/ch1#sec_introduction_dwh)) là một quá trình dẫn xuất. Giờ hãy đi xa hơn một bước. Nếu dù sao cũng phải dẫn xuất cách biểu diễn này từ cách biểu diễn khác, ta có thể chọn riêng một cách biểu diễn tối ưu cho việc ghi và những cách khác tối ưu cho việc đọc. Bạn sẽ mô hình hóa dữ liệu thế nào nếu chỉ cần tối ưu việc ghi, hoàn toàn không phải bận tâm đến hiệu quả truy vấn?

Có lẽ cách ghi dữ liệu đơn giản, nhanh và diễn đạt rõ nghĩa nhất là một *event log* (nhật ký sự kiện): mỗi lần ghi dữ liệu, ta mã hóa nó thành một chuỗi tự chứa (có thể là JSON) kèm dấu thời gian, rồi nối vào dãy sự kiện. Các sự kiện trong log là *bất biến* (*immutable*): bạn không bao giờ sửa hay xóa chúng, mà chỉ nối thêm sự kiện mới vào log (sự kiện đến sau có thể vô hiệu hóa tác dụng của sự kiện trước). Sự kiện có thể chứa thuộc tính tùy ý.

{{< xref fig="3-8" page="/ch3" anchor="fig_event_sourcing" >}}Hình 3-8{{< /xref >}} là một ví dụ có thể lấy từ hệ thống quản lý hội nghị. Quản lý hội nghị là một miền nghiệp vụ phức tạp: không chỉ từng người tham dự có thể đăng ký và trả bằng thẻ tín dụng, mà các công ty cũng có thể đặt chỗ hàng loạt, thanh toán bằng hóa đơn, rồi sau đó phân chỗ cho từng cá nhân. Diễn giả, nhà tài trợ, tình nguyện viên... có thể chiếm một số chỗ được giữ trước. Việc đặt chỗ còn có thể bị hủy; đồng thời, ban tổ chức có thể đổi sức chứa của sự kiện vì chuyển địa điểm. Khi các yếu tố này chồng lên nhau, ngay cả việc tính còn bao nhiêu chỗ trống cũng thành một truy vấn khá thách thức.

{{< fig num="3-8" id="fig_event_sourcing" src="/fig/ddia_0308.png" caption="Dùng event log bất biến làm nguồn dữ liệu có thẩm quyền, và dẫn xuất các khung nhìn vật chất hóa từ đó." class="ddia-figure ddia-figure--standard" width="1772" height="1321" />}}

Trong {{< xref fig="3-8" page="/ch3" anchor="fig_event_sourcing" >}}Hình 3-8{{< /xref >}}, mỗi thay đổi trạng thái của hội nghị (ví dụ ban tổ chức mở đăng ký, hoặc người tham dự đăng ký và hủy đăng ký) trước hết được lưu thành một sự kiện. Mỗi khi một sự kiện được nối vào log, nhiều *khung nhìn vật chất hóa* (*materialized view*, còn gọi là *phép chiếu* (*projection*) hoặc *mô hình đọc* (*read model*)) cũng được cập nhật để phản ánh tác động của sự kiện đó. Trong ví dụ hội nghị này, có thể có một khung nhìn vật chất hóa tổng hợp mọi thông tin liên quan đến trạng thái của từng lượt đặt chỗ, một khung nhìn khác tính các biểu đồ cho bảng điều khiển của ban tổ chức, và khung nhìn thứ ba tạo tệp cho máy in huy hiệu của người tham dự.

Ý tưởng dùng sự kiện làm nguồn dữ liệu có thẩm quyền và biểu diễn mọi thay đổi trạng thái thành sự kiện được gọi là *event sourcing* [^62] [^63]. Nguyên tắc duy trì các cách biểu diễn riêng, tối ưu cho đọc, và dẫn xuất chúng từ cách biểu diễn tối ưu cho ghi được gọi là *Command Query Responsibility Segregation* (CQRS) [^64]. Các thuật ngữ này xuất phát từ cộng đồng domain-driven design (DDD), dù những ý tưởng tương tự đã có từ lâu, chẳng hạn *sao chép máy trạng thái* (*state machine replication*) (xem [“Dùng log chia sẻ”](/ch10#sec_consistency_smr)).

Khi một yêu cầu từ người dùng vừa đến, nó là một *command*, và trước hết phải được kiểm tra hợp lệ. Chỉ sau khi command đã được thực thi và xác nhận là hợp lệ (ví dụ lượt đặt chỗ được yêu cầu còn đủ chỗ trống) thì nó mới trở thành một sự thật, và sự kiện tương ứng mới được nối vào log. Do đó, event log chỉ nên chứa các sự kiện hợp lệ, và những thành phần tiêu thụ event log để xây dựng khung nhìn vật chất hóa không được phép từ chối sự kiện.

Khi mô hình hóa dữ liệu theo kiểu event sourcing, nên đặt tên sự kiện ở thì quá khứ (ví dụ “chỗ đã được đặt”), vì sự kiện ghi lại một sự thật đã xảy ra. Ngay cả khi người dùng sau đó đổi hoặc hủy đặt chỗ, việc họ từng đặt chỗ vẫn là sự thật. Việc đổi hoặc hủy là một sự kiện khác được nối thêm về sau.

Event sourcing có một điểm giống với bảng sự kiện trong star schema (xem [“Star schema và snowflake schema: các schema cho phân tích”](/ch3#sec_datamodels_analytics)): cả hai đều là tập hợp các sự kiện đã xảy ra trong quá khứ. Tuy nhiên, mỗi hàng của bảng sự kiện có cùng các cột, còn trong event sourcing có thể có nhiều loại sự kiện khác nhau với các thuộc tính khác nhau. Ngoài ra, bảng sự kiện là một tập không có thứ tự, trong khi ở event sourcing thứ tự trước sau của các sự kiện rất quan trọng: nếu một lượt đặt chỗ được tạo rồi sau đó bị hủy, xử lý hai sự kiện này theo thứ tự đảo ngược thì vô nghĩa.

Event sourcing và CQRS có một số ưu điểm:

* Với nhà phát triển hệ thống, sự kiện thể hiện rõ hơn *vì sao* một việc xảy ra. Ví dụ, hiểu sự kiện “lượt đặt chỗ đã bị hủy” dễ hơn nhiều so với hiểu “cột `active` ở hàng 4001 của bảng `bookings` được đặt thành `false`, ba hàng trong bảng `seat_assignments` liên quan đến lượt đặt chỗ đó bị xóa, và một hàng biểu thị khoản hoàn tiền được chèn vào bảng `payments`”. Khi khung nhìn vật chất hóa xử lý sự kiện hủy, nó vẫn có thể sửa các hàng đó, nhưng khi các cập nhật do sự kiện thúc đẩy, lý do của chúng rõ ràng hơn nhiều.
* Một nguyên tắc then chốt của event sourcing là khung nhìn vật chất hóa phải được dẫn xuất từ event log theo cách có thể tái tạo: bạn luôn có thể xóa khung nhìn, rồi tính lại bằng cách xử lý cùng các sự kiện theo cùng thứ tự, với cùng đoạn mã. Nếu mã duy trì khung nhìn có lỗi, bạn chỉ cần xóa khung nhìn rồi tính lại bằng mã đã sửa. Lỗi cũng dễ tìm hơn, vì bạn có thể chạy đi chạy lại mã duy trì khung nhìn và kiểm tra hành vi của nó.
* Bạn có thể duy trì nhiều khung nhìn vật chất hóa, mỗi khung nhìn tối ưu cho một truy vấn cụ thể mà ứng dụng cần. Chúng có thể nằm cùng cơ sở dữ liệu với các sự kiện, hoặc trong các cơ sở dữ liệu khác nhau tùy nhu cầu. Chúng có thể dùng bất kỳ mô hình dữ liệu nào, và có thể được phi chuẩn hóa để đọc nhanh. Thậm chí bạn có thể chỉ giữ khung nhìn trong bộ nhớ và không lưu bền, miễn là khi dịch vụ khởi động lại vẫn có thể tính lại từ event log.
* Nếu quyết định trình bày thông tin hiện có theo cách mới, bạn có thể dễ dàng xây một khung nhìn vật chất hóa mới từ event log sẵn có. Hệ thống cũng có thể tiến hóa để hỗ trợ tính năng mới bằng cách thêm loại sự kiện mới, hoặc thêm thuộc tính mới vào loại sự kiện hiện có (các sự kiện cũ giữ nguyên). Bạn cũng có thể kích hoạt hành vi mới sau các sự kiện hiện có, ví dụ khi người tham dự hủy đặt chỗ thì mời chỗ đó cho người kế tiếp trong danh sách chờ.
* Nếu ghi nhầm một sự kiện, bạn có thể xóa nó rồi dựng lại các khung nhìn mà không có sự kiện đó. Ngược lại, trong cơ sở dữ liệu cập nhật và xóa dữ liệu trực tiếp, một giao dịch đã commit thường rất khó hoàn tác. Vì vậy, event sourcing giúp giảm số thao tác không thể đảo ngược trong hệ thống, khiến việc thay đổi dễ dàng hơn (xem [“Khả năng tiến hóa: giúp thay đổi dễ dàng hơn”](/ch2#sec_introduction_evolvability)).
* Event log còn có thể làm nhật ký kiểm toán, ghi lại mọi việc xảy ra trong hệ thống. Điều này có giá trị trong các ngành chịu quản lý, nơi bắt buộc phải có khả năng kiểm toán.

Tuy nhiên, event sourcing và CQRS cũng có nhược điểm:

* Phải cẩn thận khi có thông tin bên ngoài. Ví dụ, giả sử một sự kiện có giá tính bằng một loại tiền tệ, và một khung nhìn cần quy đổi nó sang loại khác. Vì tỷ giá dao động, việc lấy tỷ giá từ nguồn bên ngoài khi xử lý sự kiện sẽ có vấn đề: tính lại khung nhìn vào ngày khác có thể cho kết quả khác. Để logic xử lý sự kiện mang tính *tất định*, hoặc ghi tỷ giá vào chính sự kiện, hoặc cung cấp cách tra tỷ giá lịch sử tại dấu thời gian của sự kiện và bảo đảm cùng một dấu thời gian luôn trả về cùng một kết quả.
* Yêu cầu sự kiện bất biến gây rắc rối khi sự kiện chứa dữ liệu cá nhân của người dùng, vì người dùng có thể thực thi quyền của mình (ví dụ theo GDPR) để yêu cầu xóa dữ liệu cá nhân. Nếu mỗi người dùng có event log riêng, bạn chỉ cần xóa toàn bộ log của họ, nhưng cách này không dùng được nếu sự kiện trong log liên quan đến nhiều người dùng. Có thể thử lưu dữ liệu cá nhân bên ngoài sự kiện, hoặc mã hóa bằng một khóa để sau này xóa khóa, nhưng như vậy cũng làm việc tính lại trạng thái dẫn xuất theo yêu cầu khó hơn.
* Nếu việc xử lý sự kiện gây ra tác dụng phụ nhìn thấy được từ bên ngoài thì phải cẩn thận khi xử lý lại sự kiện. Ví dụ, có lẽ bạn không muốn gửi lại email xác nhận mỗi lần dựng lại khung nhìn vật chất hóa.

Event sourcing có thể xây trên bất kỳ cơ sở dữ liệu nào, nhưng cũng có một số hệ thống được thiết kế riêng cho mẫu này, chẳng hạn EventStoreDB, MartenDB (dựa trên PostgreSQL) và Axon Framework. Bạn cũng có thể dùng message broker như Apache Kafka để lưu event log, và dùng stream processor giữ các khung nhìn vật chất hóa luôn cập nhật. Ta sẽ quay lại các chủ đề này ở [“Change data capture và event sourcing”](/ch12#sec_stream_event_sourcing).

Yêu cầu quan trọng duy nhất là hệ thống lưu trữ sự kiện phải bảo đảm mọi khung nhìn vật chất hóa xử lý các sự kiện theo đúng thứ tự chúng xuất hiện trong log. Như ta sẽ thấy ở [Chương 10](/ch10#ch_consistency), điều này không phải lúc nào cũng dễ trong hệ thống phân tán.


## Dataframe, ma trận và mảng {#sec_datamodels_dataframes}

Các mô hình dữ liệu giới thiệu trong chương này cho đến giờ thường được dùng cho cả xử lý giao dịch lẫn phân tích (xem [“Hệ thống phân tích và hệ thống tác nghiệp”](/ch1#sec_introduction_analytics)). Còn có những mô hình dữ liệu hay gặp trong bối cảnh phân tích hoặc khoa học nhưng hiếm khi xuất hiện trong hệ thống OLTP: *dataframe* và các mảng số nhiều chiều như ma trận.

Dataframe được hỗ trợ trong R, thư viện pandas của Python, Apache Spark, ArcticDB, Dask và nhiều hệ thống khác. Các nhà khoa học dữ liệu thường dùng chúng để chuẩn bị dữ liệu huấn luyện mô hình học máy. Chúng cũng được dùng rộng rãi trong khám phá dữ liệu, phân tích thống kê và trực quan hóa dữ liệu.

Thoạt nhìn, dataframe khá giống một bảng trong cơ sở dữ liệu quan hệ, hay một bảng tính. Nó hỗ trợ một loạt thao tác hàng loạt tương tự các toán tử quan hệ: chẳng hạn áp dụng một hàm lên mọi hàng, lọc hàng theo điều kiện, nhóm theo một số cột và tổng hợp các cột khác, và nối các hàng của hai dataframe theo một khóa (trong cơ sở dữ liệu quan hệ gọi là *phép nối*, còn ở dataframe thường gọi là *merge*).

Dataframe thường không được thao tác bằng truy vấn khai báo như SQL, mà bằng một chuỗi lệnh lần lượt sửa đổi cấu trúc và nội dung của nó. Điều này khớp với quy trình làm việc điển hình của nhà khoa học dữ liệu: từng chút “nắn” dữ liệu cho đến khi nó có dạng phù hợp để trả lời câu hỏi đang xét. Những thao tác này thường chạy trên bản sao riêng của tập dữ liệu, và thường ngay trên máy cục bộ. Dù kết quả cuối cùng có thể được chia sẻ với người dùng khác.

API dataframe cung cấp nhiều thao tác vượt xa khả năng của cơ sở dữ liệu quan hệ, và cách dùng chúng thường rất khác với mô hình hóa dữ liệu quan hệ điển hình [^65]. Ví dụ, một cách dùng phổ biến của dataframe là chuyển dữ liệu từ biểu diễn kiểu mô hình quan hệ sang ma trận hoặc mảng nhiều chiều, đúng dạng đầu vào mà nhiều thuật toán học máy mong đợi.

{{< xref fig="3-9" page="/ch3" anchor="fig_dataframe_to_matrix" >}}Hình 3-9{{< /xref >}} cho một ví dụ chuyển đổi đơn giản. Bên trái là một bảng quan hệ ghi điểm (từ 1 đến 5) mà nhiều người dùng chấm cho các bộ phim; bên phải, dữ liệu được chuyển thành ma trận trong đó mỗi cột là một bộ phim và mỗi hàng là một người dùng (tương tự *bảng tổng hợp xoay* (*pivot table*) trong bảng tính). Ma trận này *thưa* (*sparse*), nghĩa là nhiều tổ hợp người dùng và phim không có dữ liệu, nhưng điều đó không sao. Ma trận có thể có hàng chục nghìn cột nên không phù hợp để đặt trong cơ sở dữ liệu quan hệ; còn dataframe và các thư viện hỗ trợ mảng thưa như NumPy của Python xử lý loại dữ liệu này dễ dàng.

{{< fig num="3-9" id="fig_dataframe_to_matrix" src="/fig/ddia_0309.png" caption="Chuyển cơ sở dữ liệu quan hệ về điểm đánh giá phim thành biểu diễn ma trận." class="ddia-figure ddia-figure--wide" width="1772" height="690" />}}

Ma trận chỉ có thể chứa số, nên cần nhiều kỹ thuật để chuyển dữ liệu phi số thành số trong ma trận. Ví dụ:

* Ngày tháng (bị lược khỏi ma trận ví dụ ở {{< xref fig="3-9" page="/ch3" anchor="fig_dataframe_to_matrix" >}}Hình 3-9{{< /xref >}}) có thể được co giãn tỷ lệ thành số thực trong một khoảng phù hợp.
* Với các cột chỉ nhận giá trị từ một tập nhỏ cố định (ví dụ thể loại phim trong cơ sở dữ liệu phim), thường dùng *mã hóa one-hot* (*one-hot encoding*): tạo một cột cho mỗi giá trị có thể có (một cột cho “hài”, một cho “chính kịch”, một cho “kinh dị”, v.v.). Với mỗi hàng đại diện cho một bộ phim, điền 1 vào cột ứng với thể loại của nó và 0 vào các cột còn lại. Cách biểu diễn này cũng dễ mở rộng cho phim thuộc nhiều thể loại.

Khi dữ liệu đã thành ma trận số, nó phù hợp cho các phép đại số tuyến tính, nền tảng của nhiều thuật toán học máy. Ví dụ, dữ liệu trong {{< xref fig="3-9" page="/ch3" anchor="fig_dataframe_to_matrix" >}}Hình 3-9{{< /xref >}} có thể dùng trong hệ thống gợi ý cho người dùng những bộ phim họ có thể thích. Dataframe đủ linh hoạt để dữ liệu dần chuyển từ dạng quan hệ sang biểu diễn ma trận, đồng thời để nhà khoa học dữ liệu tự kiểm soát cách biểu diễn nào phù hợp nhất với mục tiêu phân tích dữ liệu hoặc huấn luyện mô hình.

Cũng có những cơ sở dữ liệu chuyên lưu các mảng số nhiều chiều cỡ lớn, như TileDB [^66]. Chúng được gọi là *cơ sở dữ liệu mảng* (*array database*) và thường dùng nhất cho các tập dữ liệu khoa học, như số liệu đo địa không gian (dữ liệu raster trên lưới cách đều), ảnh y khoa hay quan sát từ kính thiên văn [^67]. Ngành tài chính cũng dùng dataframe để biểu diễn *dữ liệu chuỗi thời gian* (*time-series data*), như giá tài sản và các giao dịch theo thời gian [^68].

## Tóm tắt {#summary}

Mô hình dữ liệu là một chủ đề khổng lồ, và chương này chỉ lướt nhanh qua nhiều mô hình khác nhau. Chúng ta không đủ chỗ để đi sâu từng mô hình, nhưng hy vọng bản tổng quan này đủ gợi sự tò mò để bạn tìm hiểu thêm về mô hình phù hợp nhất với yêu cầu ứng dụng của mình.

*Mô hình quan hệ*, dù đã hơn nửa thế kỷ tuổi, vẫn là một mô hình dữ liệu quan trọng với nhiều ứng dụng — đặc biệt trong kho dữ liệu và phân tích kinh doanh, nơi star schema hay snowflake schema quan hệ và truy vấn SQL có mặt khắp nơi. Tuy nhiên, vài lựa chọn thay thế cho dữ liệu quan hệ cũng đã phổ biến ở các lĩnh vực khác:

* *Mô hình tài liệu* nhắm đến trường hợp dữ liệu là các tài liệu JSON tự chứa, và quan hệ giữa các tài liệu rất hiếm.
* *Mô hình dữ liệu đồ thị* đi theo hướng ngược lại: mọi thứ đều có thể liên quan đến mọi thứ khác, và truy vấn có thể phải đi qua nhiều bước nhảy mới tìm được dữ liệu cần quan tâm (có thể diễn đạt các truy vấn này bằng truy vấn đệ quy trong Cypher, SPARQL hoặc Datalog).
* *Dataframe* khái quát dữ liệu quan hệ cho trường hợp có rất nhiều cột, và bắc cầu giữa cơ sở dữ liệu với mảng nhiều chiều, vốn là nền tảng của nhiều tác vụ học máy, phân tích thống kê và tính toán khoa học.

Ở một mức độ nào đó, một mô hình có thể mô phỏng bằng mô hình khác — ví dụ dữ liệu đồ thị có thể biểu diễn trong cơ sở dữ liệu quan hệ — nhưng kết quả có thể rất gượng gạo, như hỗ trợ truy vấn đệ quy trong SQL đã cho thấy.

Vì vậy, người ta đã phát triển đủ loại cơ sở dữ liệu chuyên biệt cho từng mô hình dữ liệu, cung cấp ngôn ngữ truy vấn và storage engine tối ưu cho mô hình đó. Mặt khác, cơ sở dữ liệu cũng liên tục bổ sung hỗ trợ các mô hình dữ liệu khác để mở rộng sang các lĩnh vực lân cận: ví dụ, cơ sở dữ liệu quan hệ hỗ trợ dữ liệu tài liệu qua cột JSON, cơ sở dữ liệu tài liệu thêm phép nối giống mô hình quan hệ, còn hỗ trợ dữ liệu đồ thị trong SQL cũng đang dần được cải thiện.

Ta cũng đã bàn về *event sourcing*: biểu diễn dữ liệu thành append-only log gồm các sự kiện bất biến, một dạng có thể rất phù hợp để mô hình hóa hoạt động trong các miền nghiệp vụ phức tạp. Append-only log tốt cho việc ghi dữ liệu (như ta sẽ thấy ở [Chương 4](/ch4#ch_storage)). Để hỗ trợ truy vấn hiệu quả, CQRS chuyển event log thành các khung nhìn vật chất hóa tối ưu cho đọc.

Một điểm chung của các mô hình dữ liệu phi quan hệ là chúng thường không ép dữ liệu lưu trữ phải theo một schema cụ thể, nhờ đó ứng dụng dễ thích nghi hơn với yêu cầu thay đổi. Nhưng ứng dụng nhiều khả năng vẫn giả định dữ liệu có một cấu trúc nhất định. Khác biệt chỉ là schema là *tường minh* (được ép khi ghi) hay *ngầm định* (được giả định khi đọc).

Dù đã bao quát khá nhiều, vẫn còn những mô hình dữ liệu chưa được nhắc tới. Xin nêu vài ví dụ ngắn:

* Các nhà nghiên cứu bộ gen thường cần thực hiện *tìm kiếm tương đồng chuỗi* (*sequence-similarity search*), nghĩa là lấy một chuỗi rất dài (đại diện cho một phân tử DNA) và tìm các chuỗi khớp trong cơ sở dữ liệu gồm rất nhiều chuỗi tương tự nhưng không hoàn toàn giống nhau. Không cơ sở dữ liệu nào được mô tả ở đây xử lý được cách dùng này, đó là lý do các nhà nghiên cứu viết phần mềm cơ sở dữ liệu bộ gen chuyên dụng như GenBank [^69].
* Nhiều hệ thống tài chính dùng *sổ cái* (*ledger*) với kế toán ghi sổ kép làm mô hình dữ liệu. Loại dữ liệu này có thể biểu diễn trong cơ sở dữ liệu quan hệ, nhưng cũng có những cơ sở dữ liệu như TigerBeetle chuyên về mô hình này. Tiền mã hóa và blockchain thường dựa trên sổ cái phân tán, với việc chuyển giá trị được tích hợp ngay trong mô hình dữ liệu.
* *Tìm kiếm toàn văn* (*full-text search*) có thể coi là một mô hình dữ liệu thường được dùng cùng cơ sở dữ liệu. Truy xuất thông tin là một chuyên ngành lớn mà cuốn sách này sẽ không đi sâu, nhưng ta sẽ đề cập đến chỉ mục tìm kiếm (*search index*) và tìm kiếm vector ở [“Tìm kiếm toàn văn”](/ch4#sec_storage_full_text).

Tạm dừng ở đây. Ở chương tiếp theo, ta sẽ bàn một số đánh đổi khi *triển khai* các mô hình dữ liệu được mô tả trong chương này.



### Tài liệu tham khảo

[^1]: Jamie Brandon. [Unexplanations: query optimization works because sql is declarative](https://www.scattered-thoughts.net/writing/unexplanations-sql-declarative/). *scattered-thoughts.net*, February 2024. Archived at [perma.cc/P6W2-WMFZ](https://perma.cc/P6W2-WMFZ)
[^2]: Joseph M. Hellerstein. [The Declarative Imperative: Experiences and Conjectures in Distributed Logic](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2010/EECS-2010-90.pdf). Tech report UCB/EECS-2010-90, Electrical Engineering and Computer Sciences, University of California at Berkeley, June 2010. Archived at [perma.cc/K56R-VVQM](https://perma.cc/K56R-VVQM)
[^3]: Edgar F. Codd. [A Relational Model of Data for Large Shared Data Banks](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf). *Communications of the ACM*, volume 13, issue 6, pages 377–387, June 1970. [doi:10.1145/362384.362685](https://doi.org/10.1145/362384.362685)
[^4]: Michael Stonebraker and Joseph M. Hellerstein. [What Goes Around Comes Around](http://mitpress2.mit.edu/books/chapters/0262693143chapm1.pdf). In *Readings in Database Systems*, 4th edition, MIT Press, pages 2–41, 2005. ISBN: 9780262693141
[^5]: Markus Winand. [Modern SQL: Beyond Relational](https://modern-sql.com/). *modern-sql.com*, 2015. Archived at [perma.cc/D63V-WAPN](https://perma.cc/D63V-WAPN)
[^6]: Martin Fowler. [OrmHate](https://martinfowler.com/bliki/OrmHate.html). *martinfowler.com*, May 2012. Archived at [perma.cc/VCM8-PKNG](https://perma.cc/VCM8-PKNG)
[^7]: Vlad Mihalcea. [N+1 query problem with JPA and Hibernate](https://vladmihalcea.com/n-plus-1-query-problem/). *vladmihalcea.com*, January 2023. Archived at [perma.cc/79EV-TZKB](https://perma.cc/79EV-TZKB)
[^8]: Jens Schauder. [This is the Beginning of the End of the N+1 Problem: Introducing Single Query Loading](https://spring.io/blog/2023/08/31/this-is-the-beginning-of-the-end-of-the-n-1-problem-introducing-single-query). *spring.io*, August 2023. Archived at [perma.cc/6V96-R333](https://perma.cc/6V96-R333)
[^9]: William Zola. [6 Rules of Thumb for MongoDB Schema Design](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design). *mongodb.com*, June 2014. Archived at [perma.cc/T2BZ-PPJB](https://perma.cc/T2BZ-PPJB)
[^10]: Sidney Andrews and Christopher McClister. [Data modeling in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/modeling-data). *learn.microsoft.com*, February 2023. Archived at [archive.org](https://web.archive.org/web/20230207193233/https%3A//learn.microsoft.com/en-us/azure/cosmos-db/nosql/modeling-data)
[^11]: Raffi Krikorian. [Timelines at Scale](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/). At *QCon San Francisco*, November 2012. Archived at [perma.cc/V9G5-KLYK](https://perma.cc/V9G5-KLYK)
[^12]: Ralph Kimball and Margy Ross. [*The Data Warehouse Toolkit: The Definitive Guide to Dimensional Modeling*](https://learning.oreilly.com/library/view/the-data-warehouse/9781118530801/), 3rd edition. John Wiley & Sons, July 2013. ISBN: 9781118530801
[^13]: Michael Kaminsky. [Data warehouse modeling: Star schema vs. OBT](https://www.fivetran.com/blog/star-schema-vs-obt). *fivetran.com*, August 2022. Archived at [perma.cc/2PZK-BFFP](https://perma.cc/2PZK-BFFP)
[^14]: Joe Nelson. [User-defined Order in SQL](https://begriffs.com/posts/2018-03-20-user-defined-order.html). *begriffs.com*, March 2018. Archived at [perma.cc/GS3W-F7AD](https://perma.cc/GS3W-F7AD)
[^15]: Evan Wallace. [Realtime Editing of Ordered Sequences](https://www.figma.com/blog/realtime-editing-of-ordered-sequences/). *figma.com*, March 2017. Archived at [perma.cc/K6ER-CQZW](https://perma.cc/K6ER-CQZW)
[^16]: David Greenspan. [Implementing Fractional Indexing](https://observablehq.com/%40dgreensp/implementing-fractional-indexing). *observablehq.com*, October 2020. Archived at [perma.cc/5N4R-MREN](https://perma.cc/5N4R-MREN)
[^17]: Martin Fowler. [Schemaless Data Structures](https://martinfowler.com/articles/schemaless/). *martinfowler.com*, January 2013.
[^18]: Amr Awadallah. [Schema-on-Read vs. Schema-on-Write](https://www.slideshare.net/awadallah/schemaonread-vs-schemaonwrite). At *Berkeley EECS RAD Lab Retreat*, Santa Cruz, CA, May 2009. Archived at [perma.cc/DTB2-JCFR](https://perma.cc/DTB2-JCFR)
[^19]: Martin Odersky. [The Trouble with Types](https://www.infoq.com/presentations/data-types-issues/). At *Strange Loop*, September 2013. Archived at [perma.cc/85QE-PVEP](https://perma.cc/85QE-PVEP)
[^20]: Conrad Irwin. [MongoDB—Confessions of a PostgreSQL Lover](https://speakerdeck.com/conradirwin/mongodb-confessions-of-a-postgresql-lover). At *HTML5DevConf*, October 2013. Archived at [perma.cc/C2J6-3AL5](https://perma.cc/C2J6-3AL5)
[^21]: [Percona Toolkit Documentation: pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html). *docs.percona.com*, 2023. Archived at [perma.cc/9K8R-E5UH](https://perma.cc/9K8R-E5UH)
[^22]: Shlomi Noach. [gh-ost: GitHub’s Online Schema Migration Tool for MySQL](https://github.blog/2016-08-01-gh-ost-github-s-online-migration-tool-for-mysql/). *github.blog*, August 2016. Archived at [perma.cc/7XAG-XB72](https://perma.cc/7XAG-XB72)
[^23]: Shayon Mukherjee. [pg-osc: Zero downtime schema changes in PostgreSQL](https://www.shayon.dev/post/2022/47/pg-osc-zero-downtime-schema-changes-in-postgresql/). *shayon.dev*, February 2022. Archived at [perma.cc/35WN-7WMY](https://perma.cc/35WN-7WMY)
[^24]: Carlos Pérez-Aradros Herce. [Introducing pgroll: zero-downtime, reversible, schema migrations for Postgres](https://xata.io/blog/pgroll-schema-migrations-postgres). *xata.io*, October 2023. Archived at [archive.org](https://web.archive.org/web/20231008161750/https%3A//xata.io/blog/pgroll-schema-migrations-postgres)
[^25]: James C. Corbett, Jeffrey Dean, Michael Epstein, Andrew Fikes, Christopher Frost, JJ Furman, Sanjay Ghemawat, Andrey Gubarev, Christopher Heiser, Peter Hochschild, Wilson Hsieh, Sebastian Kanthak, Eugene Kogan, Hongyi Li, Alexander Lloyd, Sergey Melnik, David Mwaura, David Nagle, Sean Quinlan, Rajesh Rao, Lindsay Rolig, Dale Woodford, Yasushi Saito, Christopher Taylor, Michal Szymaniak, and Ruth Wang. [Spanner: Google’s Globally-Distributed Database](https://research.google/pubs/pub39966/). At *10th USENIX Symposium on Operating System Design and Implementation* (OSDI), October 2012.
[^26]: Donald K. Burleson. [Reduce I/O with Oracle Cluster Tables](http://www.dba-oracle.com/oracle_tip_hash_index_cluster_table.htm). *dba-oracle.com*. Archived at [perma.cc/7LBJ-9X2C](https://perma.cc/7LBJ-9X2C)
[^27]: Fay Chang, Jeffrey Dean, Sanjay Ghemawat, Wilson C. Hsieh, Deborah A. Wallach, Mike Burrows, Tushar Chandra, Andrew Fikes, and Robert E. Gruber. [Bigtable: A Distributed Storage System for Structured Data](https://research.google/pubs/pub27898/). At *7th USENIX Symposium on Operating System Design and Implementation* (OSDI), November 2006.
[^28]: Priscilla Walmsley. [*XQuery, 2nd Edition*](https://learning.oreilly.com/library/view/xquery-2nd-edition/9781491915080/). O’Reilly Media, December 2015. ISBN: 9781491915080
[^29]: Paul C. Bryan, Kris Zyp, and Mark Nottingham. [JavaScript Object Notation (JSON) Pointer](https://www.rfc-editor.org/rfc/rfc6901). RFC 6901, IETF, April 2013.
[^30]: Stefan Gössner, Glyn Normington, and Carsten Bormann. [JSONPath: Query Expressions for JSON](https://www.rfc-editor.org/rfc/rfc9535.html). RFC 9535, IETF, February 2024.
[^31]: Michael Stonebraker and Andrew Pavlo. [What Goes Around Comes Around… And Around…](https://db.cs.cmu.edu/papers/2024/whatgoesaround-sigmodrec2024.pdf). *ACM SIGMOD Record*, volume 53, issue 2, pages 21–37. [doi:10.1145/3685980.3685984](https://doi.org/10.1145/3685980.3685984)
[^32]: Lawrence Page, Sergey Brin, Rajeev Motwani, and Terry Winograd. [The PageRank Citation Ranking: Bringing Order to the Web](http://ilpubs.stanford.edu:8090/422/). Technical Report 1999-66, Stanford University InfoLab, November 1999. Archived at [perma.cc/UML9-UZHW](https://perma.cc/UML9-UZHW)
[^33]: Nathan Bronson, Zach Amsden, George Cabrera, Prasad Chakka, Peter Dimov, Hui Ding, Jack Ferris, Anthony Giardullo, Sachin Kulkarni, Harry Li, Mark Marchukov, Dmitri Petrov, Lovro Puzar, Yee Jiun Song, and Venkat Venkataramani. [TAO: Facebook’s Distributed Data Store for the Social Graph](https://www.usenix.org/conference/atc13/technical-sessions/presentation/bronson). At *USENIX Annual Technical Conference* (ATC), June 2013.
[^34]: Natasha Noy, Yuqing Gao, Anshu Jain, Anant Narayanan, Alan Patterson, and Jamie Taylor. [Industry-Scale Knowledge Graphs: Lessons and Challenges](https://cacm.acm.org/magazines/2019/8/238342-industry-scale-knowledge-graphs/fulltext). *Communications of the ACM*, volume 62, issue 8, pages 36–43, August 2019. [doi:10.1145/3331166](https://doi.org/10.1145/3331166)
[^35]: Xiyang Feng, Guodong Jin, Ziyi Chen, Chang Liu, and Semih Salihoğlu. [KÙZU Graph Database Management System](https://www.cidrdb.org/cidr2023/papers/p48-jin.pdf). At *3th Annual Conference on Innovative Data Systems Research* (CIDR 2023), January 2023.
[^36]: Maciej Besta, Emanuel Peter, Robert Gerstenberger, Marc Fischer, Michał Podstawski, Claude Barthels, Gustavo Alonso, Torsten Hoefler. [Demystifying Graph Databases: Analysis and Taxonomy of Data Organization, System Designs, and Graph Queries](https://arxiv.org/pdf/1910.09017.pdf). *arxiv.org*, October 2019.
[^37]: [Apache TinkerPop 3.6.3 Documentation](https://tinkerpop.apache.org/docs/3.6.3/reference/). *tinkerpop.apache.org*, May 2023. Archived at [perma.cc/KM7W-7PAT](https://perma.cc/KM7W-7PAT)
[^38]: Nadime Francis, Alastair Green, Paolo Guagliardo, Leonid Libkin, Tobias Lindaaker, Victor Marsault, Stefan Plantikow, Mats Rydberg, Petra Selmer, and Andrés Taylor. [Cypher: An Evolving Query Language for Property Graphs](https://doi.org/10.1145/3183713.3190657). At *International Conference on Management of Data* (SIGMOD), pages 1433–1445, May 2018. [doi:10.1145/3183713.3190657](https://doi.org/10.1145/3183713.3190657)
[^39]: Emil Eifrem. [Twitter correspondence](https://twitter.com/emileifrem/status/419107961512804352), January 2014. Archived at [perma.cc/WM4S-BW64](https://perma.cc/WM4S-BW64)
[^40]: Francesco Tisiot. [Explore the new SEARCH and CYCLE features in PostgreSQL® 14](https://aiven.io/blog/explore-the-new-search-and-cycle-features-in-postgresql-14). *aiven.io*, December 2021. Archived at [perma.cc/J6BT-83UZ](https://perma.cc/J6BT-83UZ)
[^41]: Gaurav Goel. [Understanding Hierarchies in Oracle](https://perma.cc/5ZLR-Q7EW). *towardsdatascience.com*, May 2020. Archived at [perma.cc/5ZLR-Q7EW](https://perma.cc/5ZLR-Q7EW)
[^42]: Alin Deutsch, Nadime Francis, Alastair Green, Keith Hare, Bei Li, Leonid Libkin, Tobias Lindaaker, Victor Marsault, Wim Martens, Jan Michels, Filip Murlak, Stefan Plantikow, Petra Selmer, Oskar van Rest, Hannes Voigt, Domagoj Vrgoč, Mingxi Wu, and Fred Zemke. [Graph Pattern Matching in GQL and SQL/PGQ](https://arxiv.org/abs/2112.06217). At *International Conference on Management of Data* (SIGMOD), pages 2246–2258, June 2022. [doi:10.1145/3514221.3526057](https://doi.org/10.1145/3514221.3526057)
[^43]: Alastair Green. [SQL... and now GQL](https://opencypher.org/articles/2019/09/12/SQL-and-now-GQL/). *opencypher.org*, September 2019. Archived at [perma.cc/AFB2-3SY7](https://perma.cc/AFB2-3SY7)
[^44]: Alin Deutsch, Yu Xu, and Mingxi Wu. [Seamless Syntactic and Semantic Integration of Query Primitives over Relational and Graph Data in GSQL](https://cdn2.hubspot.net/hubfs/4114546/IntegrationQuery%20PrimitivesGSQL.pdf). *tigergraph.com*, November 2018. Archived at [perma.cc/JG7J-Y35X](https://perma.cc/JG7J-Y35X)
[^45]: Oskar van Rest, Sungpack Hong, Jinha Kim, Xuming Meng, and Hassan Chafi. [PGQL: a property graph query language](https://event.cwi.nl/grades/2016/07-VanRest.pdf). At *4th International Workshop on Graph Data Management Experiences and Systems* (GRADES), June 2016. [doi:10.1145/2960414.2960421](https://doi.org/10.1145/2960414.2960421)
[^46]: Amazon Web Services. [Neptune Graph Data Model](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-data-model.html). Amazon Neptune User Guide, *docs.aws.amazon.com*. Archived at [perma.cc/CX3T-EZU9](https://perma.cc/CX3T-EZU9)
[^47]: Cognitect. [Datomic Data Model](https://docs.datomic.com/cloud/whatis/data-model.html). Datomic Cloud Documentation, *docs.datomic.com*. Archived at [perma.cc/LGM9-LEUT](https://perma.cc/LGM9-LEUT)
[^48]: David Beckett and Tim Berners-Lee. [Turtle – Terse RDF Triple Language](https://www.w3.org/TeamSubmission/turtle/). W3C Team Submission, March 2011.
[^49]: Sinclair Target. [Whatever Happened to the Semantic Web?](https://twobithistory.org/2018/05/27/semantic-web.html) *twobithistory.org*, May 2018. Archived at [perma.cc/M8GL-9KHS](https://perma.cc/M8GL-9KHS)
[^50]: Gavin Mendel-Gleason. [The Semantic Web is Dead – Long Live the Semantic Web!](https://terminusdb.com/blog/the-semantic-web-is-dead/) *terminusdb.com*, August 2022. Archived at [perma.cc/G2MZ-DSS3](https://perma.cc/G2MZ-DSS3)
[^51]: Manu Sporny. [JSON-LD and Why I Hate the Semantic Web](http://manu.sporny.org/2014/json-ld-origins-2/). *manu.sporny.org*, January 2014. Archived at [perma.cc/7PT4-PJKF](https://perma.cc/7PT4-PJKF)
[^52]: University of Michigan Library. [Biomedical Ontologies and Controlled Vocabularies](https://guides.lib.umich.edu/ontology), *guides.lib.umich.edu/ontology*. Archived at [perma.cc/Q5GA-F2N8](https://perma.cc/Q5GA-F2N8)
[^53]: Facebook. [The Open Graph protocol](https://ogp.me/), *ogp.me*. Archived at [perma.cc/C49A-GUSY](https://perma.cc/C49A-GUSY)
[^54]: Matt Haughey. [Everything you ever wanted to know about unfurling but were afraid to ask /or/ How to make your site previews look amazing in Slack](https://medium.com/slack-developer-blog/everything-you-ever-wanted-to-know-about-unfurling-but-were-afraid-to-ask-or-how-to-make-your-e64b4bb9254). *medium.com*, November 2015. Archived at [perma.cc/C7S8-4PZN](https://perma.cc/C7S8-4PZN)
[^55]: W3C RDF Working Group. [Resource Description Framework (RDF)](https://www.w3.org/RDF/). *w3.org*, February 2004.
[^56]: Steve Harris, Andy Seaborne, and Eric Prud’hommeaux. [SPARQL 1.1 Query Language](https://www.w3.org/TR/sparql11-query/). W3C Recommendation, March 2013.
[^57]: Todd J. Green, Shan Shan Huang, Boon Thau Loo, and Wenchao Zhou. [Datalog and Recursive Query Processing](http://blogs.evergreen.edu/sosw/files/2014/04/Green-Vol5-DBS-017.pdf). *Foundations and Trends in Databases*, volume 5, issue 2, pages 105–195, November 2013. [doi:10.1561/1900000017](https://doi.org/10.1561/1900000017)
[^58]: Stefano Ceri, Georg Gottlob, and Letizia Tanca. [What You Always Wanted to Know About Datalog (And Never Dared to Ask)](https://www.researchgate.net/profile/Letizia_Tanca/publication/3296132_What_you_always_wanted_to_know_about_Datalog_and_never_dared_to_ask/links/0fcfd50ca2d20473ca000000.pdf). *IEEE Transactions on Knowledge and Data Engineering*, volume 1, issue 1, pages 146–166, March 1989. [doi:10.1109/69.43410](https://doi.org/10.1109/69.43410)
[^59]: Serge Abiteboul, Richard Hull, and Victor Vianu. [*Foundations of Databases*](http://webdam.inria.fr/Alice/). Addison-Wesley, 1995. ISBN: 9780201537710, available online at [*webdam.inria.fr/Alice*](http://webdam.inria.fr/Alice/)
[^60]: Scott Meyer, Andrew Carter, and Andrew Rodriguez. [LIquid: The soul of a new graph database, Part 2](https://engineering.linkedin.com/blog/2020/liquid--the-soul-of-a-new-graph-database--part-2). *engineering.linkedin.com*, September 2020. Archived at [perma.cc/K9M4-PD6Q](https://perma.cc/K9M4-PD6Q)
[^61]: Matt Bessey. [Why, after 6 years, I’m over GraphQL](https://bessey.dev/blog/2024/05/24/why-im-over-graphql/). *bessey.dev*, May 2024. Archived at [perma.cc/2PAU-JYRA](https://perma.cc/2PAU-JYRA)
[^62]: Dominic Betts, Julián Domínguez, Grigori Melnik, Fernando Simonazzi, and Mani Subramanian. [*Exploring CQRS and Event Sourcing*](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/jj554200%28v%3Dpandp.10%29). Microsoft Patterns & Practices, July 2012. ISBN: 1621140164, archived at [perma.cc/7A39-3NM8](https://perma.cc/7A39-3NM8)
[^63]: Greg Young. [CQRS and Event Sourcing](https://www.youtube.com/watch?v=JHGkaShoyNs). At *Code on the Beach*, August 2014.
[^64]: Greg Young. [CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf). *cqrs.wordpress.com*, November 2010. Archived at [perma.cc/X5R6-R47F](https://perma.cc/X5R6-R47F)
[^65]: Devin Petersohn, Stephen Macke, Doris Xin, William Ma, Doris Lee, Xiangxi Mo, Joseph E. Gonzalez, Joseph M. Hellerstein, Anthony D. Joseph, and Aditya Parameswaran. [Towards Scalable Dataframe Systems](https://www.vldb.org/pvldb/vol13/p2033-petersohn.pdf). *Proceedings of the VLDB Endowment*, volume 13, issue 11, pages 2033–2046. [doi:10.14778/3407790.3407807](https://doi.org/10.14778/3407790.3407807)
[^66]: Stavros Papadopoulos, Kushal Datta, Samuel Madden, and Timothy Mattson. [The TileDB Array Data Storage Manager](https://www.vldb.org/pvldb/vol10/p349-papadopoulos.pdf). *Proceedings of the VLDB Endowment*, volume 10, issue 4, pages 349–360, November 2016. [doi:10.14778/3025111.3025117](https://doi.org/10.14778/3025111.3025117)
[^67]: Florin Rusu. [Multidimensional Array Data Management](https://faculty.ucmerced.edu/frusu/Papers/Report/2022-09-fntdb-arrays.pdf). *Foundations and Trends in Databases*, volume 12, numbers 2–3, pages 69–220, February 2023. [doi:10.1561/1900000069](https://doi.org/10.1561/1900000069)
[^68]: Ed Targett. [Bloomberg, Man Group team up to develop open source “ArcticDB” database](https://www.thestack.technology/bloomberg-man-group-arcticdb-database-dataframe/). *thestack.technology*, March 2023. Archived at [perma.cc/M5YD-QQYV](https://perma.cc/M5YD-QQYV)
[^69]: Dennis A. Benson, Ilene Karsch-Mizrachi, David J. Lipman, James Ostell, and David L. Wheeler. [GenBank](https://academic.oup.com/nar/article/36/suppl_1/D25/2507746). *Nucleic Acids Research*, volume 36, database issue, pages D25–D30, December 2007. [doi:10.1093/nar/gkm929](https://doi.org/10.1093/nar/gkm929) 
