---
translationKey: ddia-v2-ch5
title: Mã hóa và tiến hóa
book_kind: chapter
book_number: "5"
book_part: I
weight: 105
math: true
breadcrumbs: false
---

<a id="ch_encoding"></a>

![](/map/ch04.png)

> *Mọi thứ đều thay đổi và không gì đứng yên.*
>
> Heraclitus xứ Ephesus, được Plato dẫn trong *Cratylus* (360 TCN)

Ứng dụng chắc chắn sẽ thay đổi theo thời gian. Tính năng được thêm vào hoặc sửa đổi khi sản phẩm mới ra mắt, khi ta hiểu rõ hơn nhu cầu của người dùng, hoặc khi bối cảnh kinh doanh thay đổi. Ở [Chương 2](/ch2#ch_nonfunctional), ta đã giới thiệu khái niệm *khả năng tiến hóa* (*evolvability*): ta nên cố gắng xây dựng những hệ thống dễ thích nghi với thay đổi (xem [“Khả năng tiến hóa: giúp thay đổi dễ dàng hơn”](/ch2#sec_introduction_evolvability)).

Trong hầu hết trường hợp, thay đổi tính năng của ứng dụng cũng đòi hỏi thay đổi dữ liệu mà nó lưu trữ: có thể cần ghi nhận một trường hoặc loại bản ghi mới, hoặc cần trình bày dữ liệu hiện có theo cách mới.

Các mô hình dữ liệu ta bàn ở [Chương 3](/ch3#ch_datamodels) có những cách khác nhau để đối phó với thay đổi như vậy. Cơ sở dữ liệu quan hệ thường giả định mọi dữ liệu trong cơ sở dữ liệu đều tuân theo một schema: dù schema có thể được thay đổi (thông qua schema migration, tức các câu lệnh `ALTER`), tại mỗi thời điểm chỉ có đúng một schema có hiệu lực. Ngược lại, các cơ sở dữ liệu *schema-on-read* (hay "không có schema", *schemaless*) không áp đặt schema, nên cơ sở dữ liệu có thể chứa lẫn các định dạng dữ liệu cũ và mới được ghi vào những thời điểm khác nhau (xem [“Tính linh hoạt của schema trong mô hình tài liệu”](/ch3#sec_datamodels_schema_flexibility)).

Khi định dạng dữ liệu hoặc schema thay đổi, mã ứng dụng thường cũng cần thay đổi theo (ví dụ, bạn thêm một trường mới vào bản ghi, rồi ứng dụng bắt đầu đọc và ghi trường đó). Nhưng trong một ứng dụng lớn, thay đổi mã thường không thể diễn ra tức thì:

* Với ứng dụng phía server, có thể cần thực hiện *rolling upgrade* (*rolling upgrade*), còn gọi là *staged rollout* (*staged rollout*): triển khai phiên bản mới lên một vài node mỗi lần, kiểm tra xem phiên bản mới chạy ổn không, rồi dần dần triển khai lên tất cả node. Nhờ vậy có thể phát hành phiên bản mới mà không gây gián đoạn dịch vụ, khuyến khích phát hành thường xuyên hơn và giúp hệ thống dễ tiến hóa hơn.
* Với ứng dụng phía client, việc có nâng cấp hay không hoàn toàn tùy người dùng, và họ có thể rất lâu không cài bản cập nhật.

Điều này có nghĩa là phiên bản mã cũ và mới, cùng định dạng dữ liệu cũ và mới, có thể cùng tồn tại trong hệ thống. Để hệ thống tiếp tục chạy trơn tru, ta cần duy trì khả năng tương thích theo cả hai chiều:

*Tương thích ngược* (*backward compatibility*)
: Mã mới đọc được dữ liệu do mã cũ ghi.

*Tương thích xuôi* (*forward compatibility*)
: Mã cũ đọc được dữ liệu do mã mới ghi.

Tương thích ngược thường không khó đạt được: tác giả của mã mới biết định dạng dữ liệu do mã cũ ghi, nên có thể xử lý nó một cách tường minh (trong trường hợp cần, đơn giản là giữ lại mã cũ để đọc dữ liệu cũ). Tương thích xuôi có thể khó hơn nhiều, vì mã cũ phải bỏ qua những phần mà phiên bản mã mới thêm vào.

Tương thích xuôi còn một khó khăn nữa, như minh họa ở {{< xref fig="5-1" page="/ch5" anchor="fig_encoding_preserve_field" >}}Hình 5-1{{< /xref >}}. Giả sử bạn thêm một trường vào schema của bản ghi, và mã mới tạo ra một bản ghi chứa trường đó rồi lưu vào cơ sở dữ liệu. Sau đó, một phiên bản mã cũ, chưa biết gì về trường mới, đọc bản ghi, cập nhật nó rồi ghi lại. Trong tình huống này, ta thường muốn mã cũ giữ nguyên trường mới, dù nó không thể diễn giải trường đó. Nhưng nếu bản ghi được giải mã thành một đối tượng mô hình không giữ lại tường minh các trường không biết, dữ liệu có thể bị mất, như trong {{< xref fig="5-1" page="/ch5" anchor="fig_encoding_preserve_field" >}}Hình 5-1{{< /xref >}}.

{{< fig num="5-1" id="fig_encoding_preserve_field" src="/fig/ddia_0501.png" caption="Khi phiên bản ứng dụng cũ cập nhật dữ liệu do phiên bản mới ghi trước đó, nếu không cẩn thận, dữ liệu có thể bị mất." class="ddia-figure ddia-figure--wide" width="2880" height="1565" />}}

Chương này giới thiệu một số định dạng mã hóa dữ liệu, gồm JSON, XML, Protocol Buffers và Avro. Ta đặc biệt quan tâm đến cách chúng đối phó với thay đổi schema, và cách chúng hỗ trợ các hệ thống mà dữ liệu cũ và mới, mã cũ và mới cùng tồn tại. Sau đó, ta bàn cách các định dạng này được dùng để lưu trữ và truyền thông, gồm cơ sở dữ liệu, dịch vụ Web, REST API, lời gọi thủ tục từ xa (RPC), workflow engine, và các hệ thống hướng sự kiện như actor và message queue.

## Các định dạng mã hóa dữ liệu {#sec_encoding_formats}

Chương trình thường làm việc với dữ liệu ở (ít nhất) hai dạng:

1. Trong bộ nhớ, dữ liệu được giữ trong đối tượng, struct, danh sách, mảng, bảng băm, cây, v.v. Các cấu trúc dữ liệu này thường dùng con trỏ, và được tối ưu để CPU truy cập và thao tác hiệu quả.
2. Khi muốn ghi dữ liệu ra file hoặc gửi qua mạng, ta phải mã hóa nó thành một dãy byte tự chứa (ví dụ một tài liệu JSON). Vì con trỏ không có ý nghĩa với tiến trình khác, biểu diễn dạng dãy byte này thường trông rất khác so với cấu trúc dữ liệu trong bộ nhớ.

Do đó, ta cần một kiểu chuyển đổi giữa hai biểu diễn. Chuyển từ biểu diễn trong bộ nhớ sang dãy byte gọi là *mã hóa* (*encoding*), còn gọi là *tuần tự hóa* (*serialization*) hoặc *marshalling* (*marshalling*). Chiều ngược lại gọi là *giải mã* (*decoding*), còn gọi là *phân tích cú pháp* (*parsing*), *giải tuần tự hóa* (*deserialization*) hoặc *unmarshalling* (*unmarshalling*).


> [!TIP] Xung đột thuật ngữ
>
> Tiếc là *tuần tự hóa* cũng được dùng trong ngữ cảnh giao dịch với một nghĩa hoàn toàn khác (xem [Chương 8](/ch8#ch_transactions)). Dù "tuần tự hóa" có lẽ là thuật ngữ phổ biến hơn, để tránh nhầm lẫn giữa hai nghĩa, cuốn sách này dùng *mã hóa* xuyên suốt ở đây.


Cũng có trường hợp không cần mã hóa và giải mã. Chẳng hạn, như đã nói ở [“Thực thi truy vấn: biên dịch và vector hóa”](/ch4#sec_storage_vectorized), cơ sở dữ liệu có thể thao tác trực tiếp trên dữ liệu nén được nạp từ đĩa. Cũng có các định dạng dữ liệu *zero-copy* (*zero-copy*) như Cap’n Proto và FlatBuffers, dùng được cả khi chạy lẫn với dữ liệu trên đĩa hoặc qua mạng mà không cần bước chuyển đổi tường minh.

Tuy nhiên, hầu hết hệ thống vẫn cần chuyển đổi giữa đối tượng trong bộ nhớ và dãy byte phẳng. Đây là một vấn đề cực kỳ phổ biến, nên có vô số thư viện và định dạng mã hóa để lựa chọn. Ta hãy điểm qua một lượt.

### Định dạng riêng của từng ngôn ngữ {#id96}

Nhiều ngôn ngữ lập trình tích hợp sẵn khả năng mã hóa đối tượng trong bộ nhớ thành dãy byte. Chẳng hạn, Java có `java.io.Serializable`, Python có `pickle`, Ruby có `Marshal`, v.v. Ngoài ra còn nhiều thư viện bên thứ ba, ví dụ Kryo cho Java.

Các thư viện mã hóa này rất tiện: chỉ cần rất ít mã bổ sung là lưu và khôi phục được đối tượng trong bộ nhớ. Nhưng chúng cũng có những vấn đề sâu xa:

* Việc mã hóa thường gắn chặt với một ngôn ngữ lập trình nhất định, và rất khó đọc dữ liệu bằng ngôn ngữ khác. Nếu bạn lưu hoặc truyền dữ liệu bằng kiểu mã hóa như vậy, bạn tự ràng buộc mình vào ngôn ngữ hiện tại trong một thời gian rất dài, và cũng khó tích hợp với hệ thống của tổ chức khác (có thể dùng ngôn ngữ khác).
* Để khôi phục dữ liệu về cùng kiểu đối tượng, quá trình giải mã cần khởi tạo được các lớp tùy ý, đây thường là nguồn gốc của các lỗ hổng bảo mật [^1]. Nếu kẻ tấn công khiến ứng dụng giải mã một dãy byte tùy ý, chúng có thể khởi tạo các lớp tùy ý, từ đó thường làm được những việc nguy hiểm như thực thi mã từ xa tùy ý [^2] [^3].
* Việc quản lý phiên bản dữ liệu trong các thư viện này thường chỉ được nghĩ đến sau cùng. Vì chúng chủ yếu nhắm đến mã hóa nhanh và tiện, chúng thường bỏ qua những vấn đề rắc rối của tương thích xuôi và ngược [^4].
* Hiệu quả (thời gian CPU để mã hóa hoặc giải mã, và kích thước kết quả mã hóa) cũng thường bị xem nhẹ. Chẳng hạn, cơ chế tuần tự hóa tích hợp của Java nổi tiếng vì hiệu năng kém và mã hóa cồng kềnh [^5].

Vì vậy, dùng cách mã hóa tích hợp sẵn của ngôn ngữ thường không phải ý hay, trừ khi chỉ dùng tạm thời trong thời gian rất ngắn.

### JSON, XML và các biến thể nhị phân {#sec_encoding_json}

Khi nói đến các cách mã hóa chuẩn mà nhiều ngôn ngữ lập trình đọc và ghi được, JSON và XML là hai ứng viên nổi bật nhất. Chúng được biết đến rộng rãi, được hỗ trợ rộng rãi, và cũng "bị ghét rộng rãi" gần như ngang nhau. XML thường bị chê là quá dài dòng và phức tạp không cần thiết [^6]. JSON phổ biến chủ yếu nhờ được trình duyệt Web hỗ trợ sẵn và đơn giản hơn XML. CSV là một định dạng phổ biến khác không phụ thuộc ngôn ngữ, nhưng chỉ biểu diễn được dữ liệu dạng bảng, không có lồng nhau.

JSON, XML và CSV đều là định dạng văn bản, nên phần nào đọc được bằng mắt (dù cú pháp của chúng là chủ đề tranh cãi nóng). Ngoài những vấn đề bề mặt về cú pháp, chúng còn một số rắc rối tinh vi hơn:

* Có nhiều điểm mơ hồ trong cách mã hóa *số*. Trong XML và CSV, ta không phân biệt được một số với một chuỗi tình cờ chỉ gồm chữ số (trừ khi tham chiếu một schema bên ngoài). JSON phân biệt chuỗi và số, nhưng không phân biệt số nguyên với số dấu phẩy động, và không quy định độ chính xác.

  Điều này gây vấn đề khi xử lý số lớn. Chẳng hạn, số nguyên lớn hơn 2⁵³ không thể biểu diễn chính xác bằng số dấu phẩy động độ chính xác kép IEEE 754, nên các số này bị sai lệch khi được phân tích trong ngôn ngữ dùng số dấu phẩy động, như JavaScript [^7]. Một ví dụ thực tế là X (trước đây là Twitter) dùng số 64 bit để định danh mỗi bài đăng. JSON mà API của họ trả về chứa ID bài đăng hai lần, một lần dưới dạng số JSON và một lần dưới dạng chuỗi thập phân, để khắc phục việc ứng dụng JavaScript không phân tích đúng những số như vậy [^8].
* JSON và XML hỗ trợ tốt chuỗi Unicode (tức văn bản đọc được), nhưng không hỗ trợ chuỗi nhị phân (dãy byte không có bộ ký tự mã hóa). Chuỗi nhị phân rất hữu ích, nên người ta thường lách giới hạn này bằng cách mã hóa dữ liệu nhị phân thành văn bản bằng Base64, rồi dùng schema để chỉ rõ giá trị cần được hiểu là Base64. Cách này chạy được nhưng hơi chắp vá, và làm kích thước dữ liệu tăng 33%.
* Schema của XML và JSON rất mạnh, nên khá phức tạp để học và triển khai. Cách diễn giải đúng các dữ liệu như số và chuỗi nhị phân phụ thuộc vào thông tin trong schema, nên các ứng dụng không dùng schema XML/JSON có thể phải mã hóa cứng logic mã hóa và giải mã tương ứng.
* CSV không có schema nào, nên mỗi hàng và cột có nghĩa gì hoàn toàn do ứng dụng tự định nghĩa. Nếu ứng dụng thay đổi và thêm một hàng hoặc cột, ta phải xử lý thay đổi đó bằng tay. CSV cũng khá mơ hồ: điều gì xảy ra nếu giá trị chứa dấu phẩy hoặc ký tự xuống dòng? Dù quy tắc escape đã được đặc tả chính thức [^9], không phải bộ phân tích nào cũng cài đặt đúng.

Bất chấp những khuyết điểm này, JSON, XML và CSV vẫn đủ tốt cho nhiều mục đích. Nhiều khả năng chúng sẽ tiếp tục phổ biến, nhất là làm định dạng trao đổi dữ liệu, tức gửi dữ liệu từ tổ chức này sang tổ chức khác. Trong các trường hợp đó, miễn mọi người thống nhất được định dạng, thì định dạng đẹp hay hiệu quả thế nào thường không quan trọng. Việc khiến các tổ chức khác nhau đồng ý về *bất cứ điều gì* đã lấn át hầu hết mọi cân nhắc khác.

#### JSON Schema {#json-schema}

JSON Schema đã được áp dụng rộng rãi như một cách mô hình hóa dữ liệu khi dữ liệu được trao đổi giữa các hệ thống hoặc ghi vào kho lưu trữ. Ta gặp nó ở nhiều nơi: trong dịch vụ Web, như một phần của đặc tả OpenAPI (xem [“Dịch vụ Web”](/ch5#sec_web_services)); trong các schema registry như Confluent Schema Registry và Red Hat Apicurio Registry; và trong cơ sở dữ liệu, như tiện ích kiểm tra pg\_jsonschema của PostgreSQL và cú pháp kiểm tra `$jsonSchema` của MongoDB.

Đặc tả JSON Schema có nhiều tính năng. Nó gồm các kiểu nguyên thủy chuẩn như chuỗi, số, số nguyên, đối tượng, mảng, boolean và null, và còn có một đặc tả kiểm tra riêng để nhà phát triển gắn ràng buộc lên các trường. Chẳng hạn, một trường `port` có thể được ràng buộc nhỏ nhất là 1 và lớn nhất là 65535.

JSON Schema có thể dùng mô hình nội dung mở hoặc đóng. Mô hình nội dung mở cho phép bất kỳ trường nào không được định nghĩa trong schema xuất hiện, với kiểu dữ liệu bất kỳ, còn mô hình nội dung đóng chỉ cho phép các trường được định nghĩa tường minh. Mô hình nội dung mở được bật trong JSON Schema khi `additionalProperties` được đặt là `true`, và đó chính là giá trị mặc định. Do đó, JSON Schema thường là định nghĩa về *những gì không được phép* (tức giá trị không hợp lệ ở các trường đã định nghĩa), thay vì liệt kê *những gì được phép* trong schema.

Mô hình nội dung mở rất mạnh nhưng cũng có thể khá phức tạp. Chẳng hạn, giả sử bạn muốn định nghĩa một ánh xạ từ số nguyên (như ID) sang chuỗi. JSON không có kiểu map hay dictionary, chỉ có kiểu "đối tượng", trong đó khóa phải là chuỗi còn giá trị có thể thuộc kiểu bất kỳ. Khi đó, có thể dùng JSON Schema để ràng buộc đối tượng bằng `patternProperties` và `additionalProperties`, quy định khóa chỉ gồm chữ số và giá trị chỉ là chuỗi, như trong {{< xref eg="5-1" page="/ch5" anchor="fig_encoding_json_schema" >}}Ví dụ 5-1{{< /xref >}}.


{{< eg num="5-1" id="fig_encoding_json_schema" caption="Ví dụ JSON Schema với khóa số nguyên và giá trị chuỗi. Vì JSON Schema yêu cầu mọi khóa là chuỗi, khóa số nguyên được biểu diễn bằng chuỗi chỉ gồm chữ số." >}}
```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "patternProperties": {
        "^[0-9]+$": {
        "type": "string"
    }
    },
    "additionalProperties": false
}
```
{{< /eg >}}

Ngoài mô hình nội dung mở và đóng cùng các bộ kiểm tra, JSON Schema còn hỗ trợ logic schema có điều kiện `if/else`, kiểu có tên, tham chiếu schema từ xa, và nhiều tính năng khác. Tất cả tạo nên một ngôn ngữ schema rất mạnh, nhưng cũng khiến định nghĩa schema trở nên rườm rà, khó dùng. Việc phân giải schema từ xa, suy luận các quy tắc có điều kiện, hay tiến hóa schema theo cách tương thích xuôi hoặc ngược đều có thể rất khó [^10]. Schema XML cũng gặp vấn đề tương tự [^11].

#### Mã hóa nhị phân {#binary-encoding}

JSON ít tốn chỗ hơn XML, nhưng cả hai vẫn tốn nhiều chỗ hơn các định dạng nhị phân. Điều này dẫn đến sự ra đời của hàng loạt kiểu mã hóa nhị phân cho JSON (như MessagePack, CBOR, BSON, BJSON, UBJSON, BISON, Hessian và Smile) và cho XML (như WBXML và Fast Infoset). Các định dạng này gọn hơn và đôi khi phân tích nhanh hơn, nên được dùng trong những thị trường ngách riêng, nhưng không định dạng nào phổ biến bằng JSON và XML dạng văn bản [^12].

Một số định dạng trong số đó mở rộng tập kiểu dữ liệu, chẳng hạn phân biệt số nguyên với số dấu phẩy động, hoặc thêm hỗ trợ chuỗi nhị phân. Ngoài ra, chúng giữ nguyên mô hình dữ liệu của JSON/XML. Đặc biệt, vì không quy định schema, chúng phải đưa toàn bộ tên trường của đối tượng vào dữ liệu đã mã hóa. Nghĩa là trong mã hóa nhị phân của tài liệu JSON ở {{< xref eg="5-2" page="/ch5" anchor="fig_encoding_json" >}}Ví dụ 5-2{{< /xref >}}, ở đâu đó vẫn phải có các chuỗi `userName`, `favoriteNumber` và `interests`.

{{< eg num="5-2" id="fig_encoding_json" caption="Bản ghi ví dụ sẽ được mã hóa bằng nhiều định dạng nhị phân trong chương này" >}}
```json
{
    "userName": "Martin",
    "favoriteNumber": 1337,
    "interests": ["daydreaming", "hacking"]
}
```
{{< /eg >}}

Hãy xem MessagePack, một kiểu mã hóa nhị phân của JSON. {{< xref fig="5-2" page="/ch5" anchor="fig_encoding_messagepack" >}}Hình 5-2{{< /xref >}} cho thấy dãy byte thu được khi mã hóa tài liệu JSON trong {{< xref eg="5-2" page="/ch5" anchor="fig_encoding_json" >}}Ví dụ 5-2{{< /xref >}} bằng MessagePack. Ý nghĩa của vài byte đầu như sau:

1. Byte đầu tiên, `0x83`, cho biết phía sau là một đối tượng (bốn bit cao = `0x80`) có ba trường (bốn bit thấp = `0x03`). (Nếu đối tượng có hơn 15 trường, số trường không vừa trong bốn bit, thì nó dùng một mã nhận dạng kiểu khác và số trường được mã hóa trong hai hoặc bốn byte.)
2. Byte thứ hai, `0xa8`, cho biết phía sau là một chuỗi (bốn bit cao = `0xa0`) dài tám byte (bốn bit thấp = `0x08`).
3. Tám byte tiếp theo là tên trường `userName` mã hóa bằng ASCII. Vì độ dài đã được cho trước, không cần dấu đánh dấu hay escape để chỉ nơi chuỗi kết thúc.
4. Bảy byte kế tiếp mã hóa giá trị chuỗi `Martin` với tiền tố `0xa6` cộng sáu chữ cái, và cứ thế tiếp tục.

Mã hóa nhị phân dài 66 byte, chỉ nhỉnh hơn một chút so với 81 byte của JSON dạng văn bản đã bỏ khoảng trắng. Mọi kiểu mã hóa nhị phân của JSON đều tương tự nhau về điểm này. Mức tiết kiệm không gian ít ỏi như vậy (cùng khả năng phân tích nhanh hơn) có đáng để đánh đổi khả năng đọc được bằng mắt hay không thì khó nói.

Ở phần sau ta sẽ thấy ta làm tốt hơn nhiều: cùng bản ghi đó chỉ cần 32 byte.

{{< fig num="5-2" id="fig_encoding_messagepack" src="/fig/ddia_0502.png" caption="Bản ghi trong Ví dụ 5-2 được mã hóa bằng MessagePack." link="#fig_encoding_json" class="ddia-figure ddia-figure--standard" width="2880" height="2432" />}}


### Protocol Buffers {#sec_encoding_protobuf}

Protocol Buffers (protobuf) là một thư viện mã hóa nhị phân do Google phát triển. Nó khá giống Apache Thrift, vốn do Facebook phát triển ban đầu [^13]. Phần lớn nội dung về Protocol Buffers trong mục này cũng áp dụng được cho Thrift.

Protocol Buffers yêu cầu mọi dữ liệu cần mã hóa đều có schema. Để mã hóa dữ liệu trong {{< xref eg="5-2" page="/ch5" anchor="fig_encoding_json" >}}Ví dụ 5-2{{< /xref >}} bằng Protocol Buffers, bạn mô tả schema bằng ngôn ngữ định nghĩa giao diện (IDL) của Protocol Buffers như sau:

```protobuf
syntax = "proto3";

message Person {
    string user_name = 1;
    int64 favorite_number = 2;
    repeated string interests = 3;
}
```

Protocol Buffers đi kèm một công cụ sinh mã. Công cụ này nhận định nghĩa schema như trên và sinh ra các lớp cài đặt schema đó bằng nhiều ngôn ngữ lập trình. Ứng dụng có thể gọi mã được sinh ra để mã hóa hoặc giải mã các bản ghi theo schema. So với JSON Schema, ngôn ngữ schema của Protocol Buffers rất đơn giản: nó chỉ định nghĩa các trường của bản ghi cùng kiểu của chúng, và không hỗ trợ các ràng buộc nào khác lên giá trị của trường.

Mã hóa dữ liệu trong {{< xref eg="5-2" page="/ch5" anchor="fig_encoding_json" >}}Ví dụ 5-2{{< /xref >}} bằng bộ mã hóa Protocol Buffers cần 33 byte, như trong {{< xref fig="5-3" page="/ch5" anchor="fig_encoding_protobuf" >}}Hình 5-3{{< /xref >}} [^14].

{{< fig num="5-3" id="fig_encoding_protobuf" src="/fig/ddia_0503.png" caption="Bản ghi ví dụ được mã hóa bằng Protocol Buffers." class="ddia-figure ddia-figure--standard" width="2880" height="1775" />}}


Tương tự {{< xref fig="5-2" page="/ch5" anchor="fig_encoding_messagepack" >}}Hình 5-2{{< /xref >}}, mỗi trường có một chú thích kiểu để cho biết nó là chuỗi, số nguyên hay kiểu khác, và khi cần còn có độ dài, chẳng hạn độ dài chuỗi. Các chuỗi trong dữ liệu ("Martin", "daydreaming", "hacking") cũng được mã hóa thành ASCII như trước, chính xác hơn là UTF-8.

Khác biệt lớn nhất so với {{< xref fig="5-2" page="/ch5" anchor="fig_encoding_messagepack" >}}Hình 5-2{{< /xref >}} là không có tên trường (`userName`, `favoriteNumber`, `interests`). Thay vào đó, dữ liệu đã mã hóa chứa các *field tag* (*field tag*) dạng số (`1`, `2` và `3`), chính là các con số trong định nghĩa schema. Field tag giống như bí danh của trường: đó là cách gọn để nói ta đang đề cập đến trường nào mà không cần ghi ra tên trường.

Protocol Buffers tiết kiệm thêm không gian bằng cách nhét kiểu trường và số tag vào cùng một byte. Nó cũng dùng số nguyên có độ dài biến đổi: số 1337 được mã hóa trong hai byte, với bit cao nhất của mỗi byte cho biết còn byte nào theo sau hay không. Nghĩa là các số từ -64 đến 63 được mã hóa trong một byte, các số từ -8192 đến 8191 trong hai byte, và cứ thế. Số càng lớn thì dùng càng nhiều byte.

> [!NOTE] Ghi chú người dịch: mã hóa số nguyên có dấu
> Các khoảng có dấu ở trên áp dụng cho `sint32` và `sint64` dùng mã hóa ZigZag, không áp dụng cho `int64` trong ví dụ này. Với `int64`, giá trị không âm 0–127 được mã hóa bằng một byte dữ liệu, 128–16383 bằng hai byte, còn giá trị âm luôn chiếm mười byte dữ liệu, chưa tính field tag. Xem [tài liệu mã hóa của Protocol Buffers](https://protobuf.dev/programming-guides/encoding/#signed-integers).

Protocol Buffers không có kiểu dữ liệu danh sách hay mảng tường minh. Thay vào đó, bộ bổ nghĩa `repeated` trên trường `interests` cho biết trường này chứa một danh sách giá trị thay vì một giá trị đơn. Trong mã hóa nhị phân, các phần tử danh sách chỉ đơn giản là cùng một field tag lặp lại nhiều lần trong cùng bản ghi.

#### Field tag và tiến hóa schema {#field-tags-and-schema-evolution}

Như đã nói, schema chắc chắn sẽ thay đổi theo thời gian. Điều này gọi là *tiến hóa schema* (*schema evolution*). Protocol Buffers xử lý các thay đổi schema thế nào trong khi vẫn giữ được tương thích ngược và xuôi?

Qua ví dụ, ta thấy một bản ghi đã mã hóa chỉ là phép nối các trường đã mã hóa của nó. Mỗi trường được nhận dạng bằng số tag (các số `1`, `2`, `3` trong schema ví dụ) và được chú thích kiểu dữ liệu (ví dụ chuỗi hoặc số nguyên). Nếu một trường không có giá trị thì nó đơn giản bị bỏ khỏi bản ghi đã mã hóa. Từ đó có thể thấy field tag rất quan trọng với ý nghĩa của dữ liệu đã mã hóa. Bạn có thể đổi tên trường trong schema, vì dữ liệu đã mã hóa không bao giờ tham chiếu tên trường, nhưng không được đổi field tag của trường, vì như vậy sẽ làm mọi dữ liệu đã mã hóa hiện có mất hiệu lực.

Bạn có thể thêm trường mới vào schema, miễn là gán cho nó một số tag mới. Nếu mã cũ (không biết số tag mới) đọc dữ liệu do mã mới ghi, gồm trường mới với số tag mà nó không nhận ra, nó chỉ cần bỏ qua trường đó. Chú thích kiểu dữ liệu cho phép bộ phân tích biết cần bỏ qua bao nhiêu byte, và nó cũng nên giữ lại các trường không biết để tránh vấn đề trong {{< xref fig="5-1" page="/ch5" anchor="fig_encoding_preserve_field" >}}Hình 5-1{{< /xref >}}. Như vậy giữ được tương thích xuôi: mã cũ vẫn đọc được các bản ghi do mã mới ghi.

Còn tương thích ngược thì sao? Miễn mỗi trường có một số tag duy nhất, mã mới luôn đọc được dữ liệu cũ, vì số tag vẫn mang cùng ý nghĩa. Nếu một trường được thêm vào schema mới mà dữ liệu cũ đang đọc chưa có, nó sẽ được điền giá trị mặc định: ví dụ chuỗi rỗng cho trường chuỗi, số không cho trường số.

Xóa một trường cũng tương tự thêm trường, chỉ khác là các cân nhắc về tương thích ngược và xuôi đảo ngược lại. Ta không bao giờ được dùng lại số tag đã từng dùng, vì có thể đâu đó vẫn còn dữ liệu mang số tag cũ, và mã mới phải bỏ qua trường đó. Có thể đánh dấu số tag đã dùng là dành riêng (reserved) trong định nghĩa schema để bảo đảm sau này không quên.

Còn việc đổi kiểu dữ liệu của trường thì sao? Một số kiểu đổi được (xem chi tiết trong tài liệu), nhưng giá trị có nguy cơ bị cắt cụt. Chẳng hạn, giả sử bạn đổi số nguyên 32 bit thành 64 bit. Mã mới dễ dàng đọc dữ liệu do mã cũ ghi, vì bộ phân tích có thể điền số không vào các bit còn thiếu. Nhưng nếu mã cũ đọc dữ liệu do mã mới ghi, nó vẫn dùng biến 32 bit để giữ giá trị. Nếu giá trị 64 bit sau khi giải mã không vừa 32 bit, nó sẽ bị cắt cụt.

### Avro {#sec_encoding_avro}

Apache Avro là một định dạng mã hóa nhị phân khác, khác với Protocol Buffers theo những cách khá thú vị. Nó bắt đầu năm 2009 như một dự án con của Hadoop, vì Protocol Buffers không phù hợp với các trường hợp sử dụng của Hadoop [^15].

Avro cũng dùng schema để chỉ định cấu trúc của dữ liệu cần mã hóa. Nó có hai ngôn ngữ schema: một là Avro IDL, dành cho con người chỉnh sửa, và một dựa trên JSON, dễ đọc hơn với máy. Giống Protocol Buffers, ngôn ngữ schema của Avro chỉ xác định các trường và kiểu của chúng, và không hỗ trợ các quy tắc kiểm tra phức tạp như JSON Schema.

Schema ví dụ viết bằng Avro IDL có thể như sau:

```c
record Person {
    string                  userName;
    union { null, long }    favoriteNumber = null;
    array<string>           interests;
}
```

Biểu diễn JSON tương đương:

```c
{
    "type": "record",
    "name": "Person",
    "fields": [
        {"name": "userName",        "type": "string"},
        {"name": "favoriteNumber",  "type": ["null", "long"], "default": null},
        {"name": "interests",       "type": {"type": "array", "items": "string"}}
    ]
}
```

Trước hết, hãy lưu ý rằng schema không có số tag. Nếu ta mã hóa bản ghi ví dụ ({{< xref eg="5-2" page="/ch5" anchor="fig_encoding_json" >}}Ví dụ 5-2{{< /xref >}}) bằng schema này, mã hóa nhị phân của Avro chỉ dài 32 byte, gọn nhất trong tất cả các kiểu mã hóa ta đã xem. {{< xref fig="5-4" page="/ch5" anchor="fig_encoding_avro" >}}Hình 5-4{{< /xref >}} cho thấy dãy byte này được tạo thành như thế nào.

Nếu xem kỹ dãy byte, bạn sẽ thấy không có gì nhận dạng trường hay kiểu dữ liệu. Mã hóa chỉ đơn giản là phép nối các giá trị. Một chuỗi chỉ là tiền tố độ dài theo sau là các byte UTF-8, nhưng bản thân dữ liệu đã mã hóa không cho biết nó là chuỗi: nó cũng có thể là số nguyên hay bất cứ thứ gì khác. Số nguyên được mã hóa với độ dài biến đổi.

{{< fig num="5-4" id="fig_encoding_avro" src="/fig/ddia_0504.png" caption="Bản ghi ví dụ được mã hóa bằng Avro." class="ddia-figure ddia-figure--standard" width="2880" height="1900" />}}


Để phân tích dữ liệu nhị phân, bạn duyệt qua các trường theo thứ tự chúng xuất hiện trong schema, và dùng schema để biết kiểu dữ liệu của từng trường. Điều này có nghĩa là mã đọc chỉ giải mã đúng dữ liệu nhị phân nếu nó dùng *đúng cùng schema* với mã ghi. Bất kỳ sự không khớp nào giữa schema của bên đọc và bên ghi đều khiến dữ liệu được giải mã sai.

Vậy Avro hỗ trợ tiến hóa schema thế nào?

#### Schema của bên ghi và schema của bên đọc {#the-writers-schema-and-the-readers-schema}

Khi ứng dụng muốn mã hóa dữ liệu (để ghi vào file hay cơ sở dữ liệu, gửi qua mạng, v.v.), nó dùng phiên bản schema mà nó biết, ví dụ schema có thể được biên dịch sẵn vào ứng dụng. Đây gọi là *schema của bên ghi* (*writer’s schema*).

Khi ứng dụng muốn giải mã dữ liệu (đọc từ file hay cơ sở dữ liệu, nhận từ mạng, v.v.), nó dùng hai schema: schema của bên ghi, giống hệt schema đã dùng khi mã hóa, và *schema của bên đọc* (*reader’s schema*), có thể khác, như trong {{< xref fig="5-5" page="/ch5" anchor="fig_encoding_avro_schemas" >}}Hình 5-5{{< /xref >}}. Schema của bên đọc xác định ứng dụng mong đợi mỗi bản ghi chứa những trường nào và kiểu của chúng.

{{< fig num="5-5" id="fig_encoding_avro_schemas" src="/fig/ddia_0505.png" caption="Với Protocol Buffers, mã hóa và giải mã có thể dùng các phiên bản schema khác nhau. Với Avro, giải mã dùng hai schema: schema của bên ghi phải giống hệt schema đã dùng khi mã hóa, còn schema của bên đọc có thể là phiên bản cũ hơn hoặc mới hơn." class="ddia-figure ddia-figure--wide" width="2658" height="1269" />}}

Nếu schema của bên đọc và bên ghi giống nhau, giải mã rất đơn giản. Nếu chúng khác nhau, Avro dung hòa sự khác biệt bằng cách đặt schema của bên ghi và bên đọc cạnh nhau rồi chuyển dữ liệu từ schema đầu sang schema sau. Đặc tả Avro [^16] [^17] định nghĩa chính xác quá trình phân giải này, và được minh họa trong {{< xref fig="5-6" page="/ch5" anchor="fig_encoding_avro_resolution" >}}Hình 5-6{{< /xref >}}.

Chẳng hạn, việc thứ tự trường trong schema của bên ghi và bên đọc khác nhau không thành vấn đề, vì quá trình phân giải schema ghép các trường theo tên. Nếu mã đọc gặp một trường chỉ có trong schema của bên ghi mà không có trong schema của bên đọc, nó bỏ qua trường đó. Nếu mã đọc cần một trường mà schema của bên ghi không có trường cùng tên, nó được điền giá trị mặc định khai báo trong schema của bên đọc.

{{< fig num="5-6" id="fig_encoding_avro_resolution" src="/fig/ddia_0506.png" caption="Bộ đọc Avro dung hòa sự khác biệt giữa schema của bên ghi và schema của bên đọc." class="ddia-figure ddia-figure--wide" width="2880" height="1033" />}}

#### Quy tắc tiến hóa schema {#schema-evolution-rules}

Với Avro, tương thích xuôi nghĩa là bạn có thể có phiên bản schema mới làm bên ghi và phiên bản schema cũ làm bên đọc; ngược lại, tương thích ngược nghĩa là bạn có thể có phiên bản schema mới làm bên đọc và phiên bản cũ làm bên ghi.

Để giữ tương thích, bạn chỉ được thêm hoặc xóa trường có giá trị mặc định (trường `favoriteNumber` trong schema Avro ví dụ có giá trị mặc định `null`). Chẳng hạn, giả sử bạn thêm một trường có giá trị mặc định, nên trường mới có trong schema mới nhưng không có trong schema cũ. Khi bên đọc dùng schema mới đọc một bản ghi do schema cũ ghi, giá trị mặc định được điền cho trường còn thiếu.

Nếu bạn thêm một trường không có giá trị mặc định, bên đọc mới không đọc được dữ liệu do bên ghi cũ tạo ra, nên bạn phá vỡ tương thích ngược. Nếu bạn xóa một trường không có giá trị mặc định, bên đọc cũ không đọc được dữ liệu do bên ghi mới tạo ra, nên bạn phá vỡ tương thích xuôi.

Trong một số ngôn ngữ lập trình, `null` là giá trị mặc định chấp nhận được cho mọi biến, nhưng Avro thì không như vậy: nếu muốn cho phép một trường là `null`, bạn phải dùng *union type* (*union type*). Ví dụ, `union { null, long, string } field;` nghĩa là `field` có thể là số, chuỗi hoặc `null`. Bạn chỉ được dùng `null` làm giá trị mặc định nếu nó là nhánh đầu tiên của union. Cách này hơi dài dòng hơn việc mặc định mọi thứ đều nullable, nhưng giúp tránh lỗi bằng cách nói rõ cái gì được và không được là `null` [^18].

> [!NOTE] Ghi chú người dịch: khác biệt giữa các phiên bản Avro
> Ràng buộc về nhánh đầu tiên được nêu trong [đặc tả Avro 1.11.3](https://avro.apache.org/docs/1.11.3/specification/#unions). [Đặc tả Avro 1.12.0](https://avro.apache.org/docs/1.12.0/specification/#records) mà chương này trích dẫn đã đổi sang dùng nhánh đầu tiên của union khớp với giá trị mặc định, không yêu cầu nhánh đó phải đứng đầu union. Do đó, theo đặc tả 1.12.0, giá trị mặc định `null` chỉ cần có một nhánh `null` khớp. Ví dụ đặt nó ở vị trí đầu vẫn hợp lệ.

Đổi kiểu dữ liệu của trường là khả thi, miễn Avro chuyển đổi được kiểu. Đổi tên trường cũng được nhưng hơi phức tạp hơn: schema của bên đọc có thể chứa bí danh cho tên trường, nên nó có thể khớp tên trường trong schema cũ của bên ghi với các bí danh. Điều này có nghĩa là đổi tên trường tương thích ngược nhưng không tương thích xuôi. Tương tự, thêm một nhánh vào union type tương thích ngược nhưng không tương thích xuôi.

#### Nhưng schema của bên ghi là gì? {#but-what-is-the-writers-schema}

Đến giờ ta đã lướt qua một câu hỏi quan trọng: làm sao bên đọc biết một đoạn dữ liệu cụ thể được mã hóa bằng schema nào của bên ghi? Ta không thể nhét toàn bộ schema vào từng bản ghi, vì schema có thể lớn hơn nhiều so với dữ liệu đã mã hóa, làm mất hết lợi ích tiết kiệm không gian của mã hóa nhị phân.

Câu trả lời phụ thuộc vào ngữ cảnh dùng Avro. Một vài ví dụ:

File lớn chứa nhiều bản ghi
: Một cách dùng phổ biến của Avro là lưu file lớn chứa hàng triệu bản ghi, tất cả đều mã hóa bằng cùng một schema (ta sẽ bàn trường hợp này ở [Chương 11](/ch11#ch_batch)). Khi đó, bên ghi file chỉ cần ghi schema của bên ghi một lần ở đầu file. Avro quy định một định dạng file cho việc này, gọi là object container file.

Cơ sở dữ liệu có từng bản ghi được ghi riêng
: Trong cơ sở dữ liệu, các bản ghi khác nhau có thể được ghi vào những thời điểm khác nhau bằng các schema của bên ghi khác nhau, và ta không thể giả định mọi bản ghi đều dùng cùng một schema. Giải pháp đơn giản nhất là đặt một số phiên bản ở đầu mỗi bản ghi đã mã hóa, và giữ một danh sách các phiên bản schema trong cơ sở dữ liệu. Bên đọc lấy một bản ghi, trích số phiên bản, rồi lấy từ cơ sở dữ liệu schema của bên ghi ứng với phiên bản đó, và dùng nó để giải mã phần còn lại của bản ghi.

  Ví dụ, Confluent Schema Registry của Apache Kafka [^19] và Espresso của LinkedIn [^20] hoạt động theo cách này.

Gửi bản ghi qua kết nối mạng
: Khi hai tiến trình giao tiếp qua kết nối mạng hai chiều, chúng có thể thương lượng phiên bản schema khi thiết lập kết nối, rồi dùng schema đó trong suốt vòng đời của kết nối. Giao thức Avro RPC (xem [“Luồng dữ liệu qua dịch vụ: REST và RPC”](/ch5#sec_encoding_dataflow_rpc)) hoạt động như vậy.

Dù theo cách nào, một cơ sở dữ liệu các phiên bản schema vẫn rất hữu ích, vì nó vừa là tài liệu vừa cho bạn cơ hội kiểm tra tính tương thích của schema [^21]. Số phiên bản có thể là một số nguyên tăng dần đơn giản, hoặc là giá trị băm của schema.

#### Schema sinh động {#dynamically-generated-schemas}

Một ưu điểm của Avro so với Protocol Buffers là schema không chứa số tag nào. Nhưng điều này có gì quan trọng? Giữ vài con số trong schema thì có vấn đề gì?

Khác biệt là Avro thân thiện hơn với các schema *sinh động* (được sinh ra tự động). Chẳng hạn, giả sử bạn có một cơ sở dữ liệu quan hệ và muốn kết xuất nội dung của nó ra file, dùng định dạng nhị phân để tránh các vấn đề của các định dạng văn bản (JSON, CSV, XML). Nếu dùng Avro, bạn có thể dễ dàng sinh một schema Avro (theo biểu diễn JSON đã thấy ở trên) từ schema quan hệ và mã hóa nội dung cơ sở dữ liệu bằng schema đó, kết xuất toàn bộ vào một object container file của Avro [^22]. Bạn sinh một schema bản ghi cho mỗi bảng trong cơ sở dữ liệu, và mỗi cột trở thành một trường trong bản ghi. Tên cột trong cơ sở dữ liệu được ánh xạ sang tên trường trong Avro.

Giờ nếu schema cơ sở dữ liệu thay đổi (ví dụ một bảng thêm một cột và bớt một cột), bạn chỉ cần sinh schema Avro mới từ schema cơ sở dữ liệu đã cập nhật và xuất dữ liệu bằng schema mới. Quá trình xuất dữ liệu không cần để ý đến thay đổi schema cụ thể nào: nó chỉ làm lại phép chuyển đổi schema mỗi lần chạy. Ai đọc file dữ liệu mới sẽ thấy các trường của bản ghi đã đổi, nhưng vì các trường được nhận dạng theo tên, schema của bên ghi đã cập nhật vẫn khớp được với schema cũ của bên đọc.

Ngược lại, nếu dùng Protocol Buffers cho việc này, field tag có lẽ phải được gán thủ công: mỗi khi schema cơ sở dữ liệu thay đổi, quản trị viên phải cập nhật thủ công ánh xạ từ tên cột sang field tag. (Có thể tự động hóa việc này, nhưng bộ sinh schema phải hết sức cẩn thận để không gán lại field tag đã dùng trước đó.) Sinh schema động chưa bao giờ là mục tiêu thiết kế của Protocol Buffers, còn với Avro thì có.

### Ưu điểm của schema {#sec_encoding_schemas}

Như ta đã thấy, Protocol Buffers và Avro đều dùng schema để mô tả định dạng mã hóa nhị phân. Ngôn ngữ schema của chúng đơn giản hơn nhiều so với schema XML hay JSON Schema, vốn hỗ trợ các quy tắc kiểm tra chi tiết hơn (ví dụ "giá trị chuỗi của trường này phải khớp một biểu thức chính quy" hoặc "giá trị số nguyên của trường này phải nằm giữa 0 và 100"). Vì Protocol Buffers và Avro cài đặt đơn giản hơn và dùng cũng đơn giản hơn, chúng đã được hỗ trợ khá rộng rãi trong nhiều ngôn ngữ lập trình.

Những ý tưởng đằng sau các kiểu mã hóa này hoàn toàn không mới. Chẳng hạn, chúng có nhiều điểm chung với ASN.1, một ngôn ngữ định nghĩa schema được chuẩn hóa lần đầu năm 1984 [^23] [^24]. Nó từng được dùng để định nghĩa nhiều giao thức mạng, và mã hóa nhị phân DER của nó vẫn được dùng để mã hóa chứng chỉ SSL (X.509) [^25]. Giống Protocol Buffers, ASN.1 cũng hỗ trợ tiến hóa schema bằng số tag [^26]. Tuy nhiên, ASN.1 rất phức tạp và tài liệu kém, nên có lẽ không phù hợp cho ứng dụng mới.

Nhiều hệ thống dữ liệu cũng cài đặt một kiểu mã hóa nhị phân độc quyền cho dữ liệu của chúng. Chẳng hạn, hầu hết cơ sở dữ liệu quan hệ có giao thức mạng riêng để gửi truy vấn và nhận phản hồi. Các giao thức đó thường dành riêng cho từng cơ sở dữ liệu, và nhà cung cấp cơ sở dữ liệu cung cấp trình điều khiển (ví dụ dùng API ODBC hoặc JDBC) để giải mã phản hồi từ giao thức mạng của cơ sở dữ liệu thành cấu trúc dữ liệu trong bộ nhớ.

Vậy, dù các định dạng văn bản như JSON, XML và CSV rất phổ biến, các kiểu mã hóa nhị phân dựa trên schema cũng là một lựa chọn khả thi, và có nhiều tính chất tốt:

* Chúng có thể gọn hơn nhiều so với các biến thể "JSON nhị phân", vì không cần đưa tên trường vào dữ liệu đã mã hóa.
* Schema là một dạng tài liệu rất có giá trị: vì schema bắt buộc phải có để giải mã, bạn có thể chắc chắn nó luôn cập nhật (trong khi tài liệu viết tay dễ bị lệch khỏi thực tế).
* Việc giữ cơ sở dữ liệu schema cho phép bạn kiểm tra tính tương thích xuôi và ngược của các thay đổi schema trước khi triển khai bất kỳ thứ gì.
* Với người dùng các ngôn ngữ lập trình kiểu tĩnh, sinh mã từ schema rất hữu ích, vì nó cho phép kiểm tra kiểu ngay lúc biên dịch.

Tóm lại, tiến hóa schema cho phép đạt cùng mức linh hoạt như cơ sở dữ liệu JSON không có schema/schema-on-read (xem [“Tính linh hoạt của schema trong mô hình tài liệu”](/ch3#sec_datamodels_schema_flexibility)), đồng thời đem lại những đảm bảo tốt hơn về dữ liệu và công cụ tốt hơn.

## Các chế độ luồng dữ liệu {#sec_encoding_dataflow}

Ở đầu chương này ta đã nói rằng bất cứ khi nào muốn gửi dữ liệu cho một tiến trình khác không chia sẻ bộ nhớ (ví dụ gửi qua mạng hoặc ghi ra file), ta cần mã hóa nó thành một dãy byte. Sau đó ta đã bàn nhiều cách mã hóa để làm việc đó.

Ta cũng đã bàn về tương thích xuôi và ngược, vốn quan trọng với khả năng tiến hóa: chúng giúp thay đổi dễ dàng hơn bằng cách cho phép nâng cấp độc lập các phần của hệ thống, mà không phải thay đổi mọi thứ cùng lúc. Tính tương thích là mối quan hệ giữa tiến trình mã hóa dữ liệu và tiến trình giải mã dữ liệu.

Đây là một khái niệm khá trừu tượng, vì dữ liệu có thể chảy từ tiến trình này sang tiến trình khác theo nhiều cách. Ai mã hóa, ai giải mã? Phần còn lại của chương này khám phá một số cách phổ biến nhất để dữ liệu chảy giữa các tiến trình:

* Qua cơ sở dữ liệu (xem [“Luồng dữ liệu qua cơ sở dữ liệu”](/ch5#sec_encoding_dataflow_db))
* Qua lời gọi dịch vụ (xem [“Luồng dữ liệu qua dịch vụ: REST và RPC”](/ch5#sec_encoding_dataflow_rpc))
* Qua workflow engine (xem [“Durable execution và workflow”](/ch5#sec_encoding_dataflow_workflows))
* Qua message bất đồng bộ (xem [“Kiến trúc hướng sự kiện”](/ch5#sec_encoding_dataflow_msg))

### Luồng dữ liệu qua cơ sở dữ liệu {#sec_encoding_dataflow_db}

Trong cơ sở dữ liệu, tiến trình ghi vào cơ sở dữ liệu mã hóa dữ liệu, còn tiến trình đọc từ cơ sở dữ liệu giải mã nó. Có thể chỉ có một tiến trình truy cập cơ sở dữ liệu, khi đó bên đọc đơn giản là phiên bản sau của chính tiến trình đó, và bạn có thể xem việc lưu thứ gì đó vào cơ sở dữ liệu như *gửi một thông điệp cho chính mình trong tương lai*.

Rõ ràng ở đây cần tương thích ngược, nếu không bản thân bạn trong tương lai sẽ không giải mã được thứ bạn đã ghi trước đó.

Nói chung, việc nhiều tiến trình khác nhau cùng truy cập một cơ sở dữ liệu là chuyện thường. Các tiến trình đó có thể thuộc những ứng dụng hoặc dịch vụ khác nhau, hoặc chỉ là nhiều instance của cùng một dịch vụ chạy song song để đảm bảo khả năng mở rộng hay chịu lỗi. Dù theo cách nào, trong một ứng dụng đang thay đổi, rất có thể một số tiến trình truy cập cơ sở dữ liệu đang chạy mã mới và một số vẫn chạy mã cũ. Chẳng hạn, trong lúc rolling upgrade, một số instance đã được cập nhật còn số khác thì chưa.

Điều này có nghĩa là một giá trị trong cơ sở dữ liệu có thể được ghi bởi phiên bản mã *mới hơn*, rồi sau đó được đọc bởi phiên bản mã *cũ hơn* vẫn đang chạy. Do đó, cơ sở dữ liệu cũng thường cần tương thích xuôi.

#### Các giá trị khác nhau được ghi vào những thời điểm khác nhau {#different-values-written-at-different-times}

Cơ sở dữ liệu thường cho phép cập nhật bất kỳ giá trị nào vào bất cứ lúc nào. Điều này có nghĩa là trong một cơ sở dữ liệu, có thể có giá trị vừa được ghi cách đây năm mili giây, và cũng có giá trị đã được ghi cách đây năm năm.

Khi bạn triển khai phiên bản mới của ứng dụng (ít nhất với ứng dụng phía server), bạn có thể thay thế hoàn toàn phiên bản cũ bằng phiên bản mới chỉ trong vài phút. Nhưng với nội dung cơ sở dữ liệu thì không: trừ khi được ghi lại tường minh, dữ liệu năm năm tuổi vẫn nằm đó với mã hóa ban đầu. Hiện tượng này đôi khi được tóm gọn là *dữ liệu sống lâu hơn mã*.

Tất nhiên, ta có thể ghi lại dữ liệu (tức *di trú* nó) sang schema mới, nhưng với tập dữ liệu lớn việc này rất tốn kém, nên hầu hết cơ sở dữ liệu cố tránh. Hầu hết cơ sở dữ liệu quan hệ cho phép một số thay đổi schema đơn giản, như thêm một cột mới với giá trị mặc định `null`, mà không cần ghi lại dữ liệu hiện có. Khi đọc một hàng cũ, nếu dữ liệu mã hóa trên đĩa thiếu một cột nào đó, cơ sở dữ liệu điền `null` cho cột đó.

Vì vậy, tiến hóa schema khiến toàn bộ cơ sở dữ liệu trông như được mã hóa bằng một schema duy nhất, dù kho lưu trữ bên dưới có thể chứa các bản ghi được mã hóa bằng nhiều phiên bản schema lịch sử khác nhau.

Những thay đổi schema phức tạp hơn, chẳng hạn đổi thuộc tính đơn trị thành đa trị, hoặc chuyển một phần dữ liệu sang bảng khác, vẫn đòi hỏi ghi lại dữ liệu, thường ở tầng ứng dụng [^27]. Duy trì tương thích xuôi và ngược qua các lần di trú như vậy vẫn là một vấn đề nghiên cứu [^28].

#### Lưu trữ dài hạn {#archival-storage}

Có thể thỉnh thoảng bạn chụp snapshot cơ sở dữ liệu, chẳng hạn để sao lưu hoặc nạp vào kho dữ liệu (xem [“Kho dữ liệu”](/ch1#sec_introduction_dwh)). Trong trường hợp này, bản kết xuất dữ liệu thường được mã hóa bằng schema mới nhất, dù cơ sở dữ liệu nguồn ban đầu chứa lẫn nhiều phiên bản schema từ các thời kỳ khác nhau. Dù sao ta cũng đang chép dữ liệu ra một bản mới, nên cứ mã hóa bản đó một cách nhất quán.

Vì bản kết xuất dữ liệu được ghi một lần và sau đó không đổi, các định dạng như object container file của Avro rất phù hợp. Đây cũng là dịp tốt để mã hóa dữ liệu theo định dạng cột phù hợp cho phân tích, như Parquet (xem [“Nén cột”](/ch4#sec_storage_column_compression)).

Ở [Chương 11](/ch11#ch_batch), ta sẽ bàn thêm về cách dùng dữ liệu trong lưu trữ dài hạn.

### Luồng dữ liệu qua dịch vụ: REST và RPC {#sec_encoding_dataflow_rpc}

Khi cần giao tiếp qua mạng giữa các tiến trình, có vài cách tổ chức khác nhau. Cách phổ biến nhất có hai vai trò: *client* (*client*) và *server* (*server*). Server cung cấp một API qua mạng, còn client kết nối tới server để gửi yêu cầu đến API đó. API mà server cung cấp được gọi là *dịch vụ* (*service*).

Web hoạt động theo cách này: client (trình duyệt Web) gửi yêu cầu tới Web server, dùng yêu cầu `GET` để tải HTML, CSS, JavaScript, hình ảnh, v.v., và dùng yêu cầu `POST` để gửi dữ liệu lên server. API này gồm một tập giao thức và định dạng dữ liệu chuẩn hóa (HTTP, URL, SSL/TLS, HTML, v.v.). Vì trình duyệt Web, Web server và tác giả website hầu hết đều tuân theo các chuẩn này, về lý thuyết bạn có thể dùng bất kỳ trình duyệt nào để truy cập bất kỳ website nào.

Trình duyệt Web không phải loại client duy nhất. Chẳng hạn, ứng dụng native chạy trên thiết bị di động hoặc máy tính để bàn cũng thường giao tiếp với server, và ứng dụng JavaScript phía client chạy trong trình duyệt cũng có thể gửi yêu cầu HTTP. Trong trường hợp này, server trả về không phải HTML cho người đọc mà là dữ liệu đã mã hóa, thường là JSON, để mã phía client xử lý tiếp. Dù HTTP có thể được dùng làm giao thức truyền tải, API xây dựng trên nó vẫn do ứng dụng tự định nghĩa, và client với server phải thống nhất về chi tiết của API đó.

Ở một số khía cạnh, dịch vụ giống cơ sở dữ liệu: chúng thường cho phép client gửi và truy vấn dữ liệu. Tuy nhiên, trong khi cơ sở dữ liệu cho phép truy vấn tùy ý bằng các ngôn ngữ truy vấn ta đã bàn ở [Chương 3](/ch3#ch_datamodels), dịch vụ chỉ cung cấp một API riêng cho ứng dụng, chỉ nhận các đầu vào và chỉ tạo ra các đầu ra được định sẵn bởi logic nghiệp vụ (mã ứng dụng) của dịch vụ [^29]. Hạn chế này mang lại một mức đóng gói: dịch vụ có thể ràng buộc chi tiết việc client được làm và không được làm gì.

Một mục tiêu thiết kế then chốt của kiến trúc hướng dịch vụ hay microservices là giúp ứng dụng dễ thay đổi và bảo trì hơn bằng cách cho phép triển khai và tiến hóa các dịch vụ một cách độc lập. Một nguyên tắc phổ biến là mỗi dịch vụ do một nhóm sở hữu, và nhóm đó có thể phát hành phiên bản mới của dịch vụ thường xuyên mà không cần phối hợp với các nhóm khác. Do đó ta kỳ vọng phiên bản cũ và mới của server và client chạy đồng thời, nên mã hóa dữ liệu mà server và client dùng phải tương thích giữa các phiên bản của API dịch vụ.

#### Dịch vụ Web {#sec_web_services}

Khi dùng HTTP làm giao thức nền để giao tiếp với dịch vụ, nó được gọi là *dịch vụ Web* (*Web service*). Dịch vụ Web thường được dùng để xây dựng kiến trúc hướng dịch vụ hoặc microservices (đã bàn ở [“Microservices và serverless”](/ch1#sec_introduction_microservices)). Tuy nhiên, cái tên "dịch vụ Web" hơi sai lệch, vì nó không chỉ dùng trên Web mà còn xuất hiện trong vài bối cảnh khác. Chẳng hạn:

1. Ứng dụng client chạy trên thiết bị của người dùng (ví dụ ứng dụng native trên di động, hoặc ứng dụng Web JavaScript trong trình duyệt) gửi yêu cầu HTTP tới một dịch vụ. Các yêu cầu này thường đi qua Internet công cộng.
2. Trong kiến trúc hướng dịch vụ hoặc microservices, một dịch vụ gửi yêu cầu tới dịch vụ khác do cùng tổ chức sở hữu, thường đặt trong cùng một trung tâm dữ liệu.
3. Một dịch vụ gửi yêu cầu tới dịch vụ do tổ chức khác sở hữu, thường qua Internet. Cách này dùng để trao đổi dữ liệu giữa hệ thống backend của các tổ chức khác nhau, gồm các API công khai do dịch vụ trực tuyến cung cấp, chẳng hạn hệ thống thanh toán thẻ tín dụng, và OAuth để chia sẻ quyền truy cập dữ liệu người dùng.

Triết lý thiết kế dịch vụ phổ biến nhất là REST, xây dựng trên các nguyên tắc của HTTP [^30] [^31]. REST nhấn mạnh định dạng dữ liệu đơn giản, dùng URL để định danh tài nguyên, và tận dụng các tính năng của HTTP để kiểm soát cache, xác thực và thương lượng kiểu nội dung. API thiết kế theo nguyên tắc REST được gọi là API *RESTful*.

Mã gọi API của dịch vụ Web phải biết cần truy vấn endpoint HTTP nào, gửi dữ liệu định dạng ra sao, và mong đợi phản hồi thế nào. Ngay cả khi dịch vụ tuân theo nguyên tắc thiết kế RESTful, client vẫn phải biết những chi tiết này bằng cách nào đó. Nhà phát triển dịch vụ thường dùng ngôn ngữ định nghĩa giao diện (IDL) để định nghĩa và tài liệu hóa các endpoint API và mô hình dữ liệu, rồi tiến hóa chúng dần theo thời gian. Nhà phát triển khác có thể dựa vào định nghĩa dịch vụ để biết cách gửi yêu cầu. Hai IDL dịch vụ phổ biến nhất là OpenAPI (còn gọi là Swagger [^32]) và gRPC. OpenAPI dùng cho dịch vụ Web gửi và nhận dữ liệu JSON, còn dịch vụ gRPC gửi và nhận dữ liệu Protocol Buffers.

Nhà phát triển thường viết định nghĩa dịch vụ OpenAPI bằng JSON hoặc YAML, xem {{< xref eg="5-3" page="/ch5" anchor="fig_open_api_def" >}}Ví dụ 5-3{{< /xref >}}. Định nghĩa dịch vụ có thể mô tả endpoint, tài liệu, phiên bản, mô hình dữ liệu và nhiều thứ khác. Định nghĩa gRPC trông tương tự, nhưng dùng cú pháp định nghĩa dịch vụ của Protocol Buffers.

{{< eg num="5-3" id="fig_open_api_def" caption="Ví dụ định nghĩa dịch vụ OpenAPI viết bằng YAML" >}}
```yaml
openapi: 3.0.0
info:
  title: Ping, Pong
  version: 1.0.0
servers:
  - url: http://localhost:8080
paths:
  /ping:
    get:
      summary: Given a ping, returns a pong message
      responses:
        '200':
          description: A pong
          content:
            application/json:
              schema:
                type: object
                properties:
                  message:
                    type: string
                    example: Pong!
```
{{< /eg >}}

Ngay cả khi đã chọn triết lý thiết kế và IDL, nhà phát triển vẫn phải viết mã để cài đặt các lời gọi API của dịch vụ. Thường dùng một framework dịch vụ để đơn giản hóa việc này. Các framework như Spring Boot, FastAPI và gRPC cho phép nhà phát triển chỉ cần viết logic nghiệp vụ cho từng endpoint API, còn framework lo định tuyến, số liệu đo, cache, xác thực, v.v. {{< xref eg="5-4" page="/ch5" anchor="fig_fastapi_def" >}}Ví dụ 5-4{{< /xref >}} là một cài đặt Python của dịch vụ được định nghĩa trong {{< xref eg="5-3" page="/ch5" anchor="fig_open_api_def" >}}Ví dụ 5-3{{< /xref >}}.

{{< eg num="5-4" id="fig_fastapi_def" caption="Cài đặt dịch vụ định nghĩa trong Ví dụ 5-3 bằng FastAPI" >}}
```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="Ping, Pong", version="1.0.0")

class PongResponse(BaseModel):
    message: str = "Pong!"

@app.get("/ping", response_model=PongResponse,
         summary="Given a ping, returns a pong message")
async def ping():
    return PongResponse()
```
{{< /eg >}}

Nhiều framework kết hợp định nghĩa dịch vụ với mã server. Chẳng hạn, trong FastAPI (một framework Python phổ biến), nhà phát triển viết server bằng mã và framework tự sinh IDL. Các framework như gRPC thì ngược lại, viết định nghĩa dịch vụ trước rồi sinh khung mã server. Cả hai cách đều có thể sinh thư viện client và SDK cho nhiều ngôn ngữ từ định nghĩa dịch vụ. Ngoài sinh mã, các công cụ IDL như Swagger còn có thể sinh tài liệu, kiểm tra tính tương thích của các thay đổi schema, và cung cấp giao diện đồ họa để nhà phát triển truy vấn và thử nghiệm dịch vụ.

#### Vấn đề của lời gọi thủ tục từ xa (RPC) {#sec_problems_with_rpc}

Dịch vụ Web chỉ là hiện thân mới nhất của một dòng công nghệ gửi yêu cầu API qua mạng. Nhiều công nghệ trước đó từng được thổi phồng nhưng có những vấn đề nghiêm trọng: Enterprise JavaBeans (EJB) và Remote Method Invocation (RMI) của Java bị giới hạn trong Java. Distributed Component Object Model (DCOM) bị giới hạn trong nền tảng Microsoft. Common Object Request Broker Architecture (CORBA) quá phức tạp và không hỗ trợ tương thích ngược hay xuôi [^33]. SOAP và các framework dịch vụ Web WS-\* nhắm đến khả năng tương tác giữa các nhà cung cấp, nhưng cũng bị vướng độ phức tạp và vấn đề tương thích [^34] [^35] [^36].

Tất cả các công nghệ này đều dựa trên ý tưởng *lời gọi thủ tục từ xa* (*remote procedure call*), viết tắt RPC, vốn đã có từ thập niên 1970 [^37]. Mô hình RPC cố làm cho việc gửi yêu cầu tới một dịch vụ mạng từ xa trông giống hệt việc gọi một hàm hoặc phương thức trong ngôn ngữ lập trình ngay trong cùng tiến trình (cách trừu tượng hóa này gọi là *tính trong suốt về vị trí* (*location transparency*)). RPC thoạt nhìn rất tiện, nhưng cách tiếp cận này có khiếm khuyết căn bản [^38] [^39]. Yêu cầu mạng khác rất nhiều so với lời gọi hàm cục bộ:

* Lời gọi hàm cục bộ có thể dự đoán được và thành công hay thất bại chỉ phụ thuộc vào các tham số bạn kiểm soát. Yêu cầu mạng thì không thể dự đoán: yêu cầu hoặc phản hồi có thể bị mất do sự cố mạng, hoặc máy ở xa có thể chậm hay không khả dụng, và những chuyện đó hoàn toàn nằm ngoài tầm kiểm soát của bạn. Sự cố mạng là chuyện thường, nên bạn phải lường trước, chẳng hạn bằng cách thử lại các yêu cầu thất bại.
* Lời gọi hàm cục bộ hoặc trả về kết quả, hoặc ném ngoại lệ, hoặc không bao giờ trả về (do vòng lặp vô hạn hay tiến trình sập). Yêu cầu mạng còn một kết quả khả dĩ khác: nó có thể trả về mà không có kết quả, do *timeout* (*timeout*). Khi đó bạn đơn giản không biết chuyện gì đã xảy ra: nếu không nhận được phản hồi từ dịch vụ từ xa, bạn không có cách nào biết yêu cầu đã đến nơi hay chưa ([Chương 9](/ch9#ch_distributed) sẽ bàn kỹ vấn đề này).
* Nếu bạn thử lại một yêu cầu mạng thất bại, có thể yêu cầu gốc thực ra đã thành công và chỉ có phản hồi bị mất. Khi đó, thử lại sẽ khiến thao tác bị thực hiện nhiều lần, trừ khi bạn xây dựng vào giao thức một cơ chế loại bỏ trùng lặp, gọi là *idempotence* (*idempotence*) [^40]. Lời gọi hàm cục bộ không có vấn đề này (xem [“Idempotence”](/ch12#sec_stream_idempotence)).
* Lời gọi hàm cục bộ thường mất thời gian gần như nhau mỗi lần. Yêu cầu mạng không chỉ chậm hơn nhiều so với lời gọi hàm mà độ trễ còn biến động mạnh: trong những lúc thuận lợi có thể xong trong chưa đầy một mili giây, nhưng khi mạng tắc nghẽn hoặc dịch vụ từ xa quá tải, cùng một thao tác có thể mất vài giây.
* Khi gọi hàm cục bộ, bạn có thể truyền hiệu quả tham chiếu (con trỏ) tới các đối tượng trong bộ nhớ cục bộ. Khi gửi yêu cầu mạng, mọi tham số phải được mã hóa thành dãy byte có thể gửi qua mạng. Với các giá trị nguyên thủy bất biến như số hay chuỗi ngắn thì không sao, nhưng sẽ nhanh chóng rắc rối khi dữ liệu lớn hơn hoặc có đối tượng khả biến.
* Client và dịch vụ có thể được cài đặt bằng các ngôn ngữ lập trình khác nhau, nên framework RPC phải chuyển đổi kiểu dữ liệu giữa các ngôn ngữ. Điều này có thể gây ra hậu quả xấu xí, vì không phải ngôn ngữ nào cũng có cùng kiểu: chẳng hạn ta đã nhắc rằng JavaScript không biểu diễn chính xác số nguyên lớn hơn 2⁵³ (xem [“JSON, XML và các biến thể nhị phân”](/ch5#sec_encoding_json)). Vấn đề này không tồn tại trong một tiến trình duy nhất chỉ dùng một ngôn ngữ.

Tất cả những khác biệt này cho thấy không có lý do gì để cố làm dịch vụ từ xa trông như đối tượng cục bộ trong ngôn ngữ lập trình, vì chúng về bản chất là hai thứ khác nhau. Một phần sức hấp dẫn của REST là nó coi việc chuyển trạng thái qua mạng là một quá trình khác với lời gọi hàm.

#### Bộ cân bằng tải, service discovery và service mesh {#sec_encoding_service_discovery}

Mọi dịch vụ đều giao tiếp qua mạng, nên client phải biết địa chỉ của dịch vụ đích, một vấn đề gọi là *service discovery* (*service discovery*). Cách đơn giản nhất là cấu hình client với địa chỉ IP và cổng nơi dịch vụ chạy. Cách này dùng được, nhưng nếu server ngoại tuyến, chuyển sang máy khác hoặc bị quá tải, ta phải cấu hình lại client thủ công.

Để cải thiện tính sẵn sàng và khả năng mở rộng, thường chạy nhiều instance của một dịch vụ trên các máy khác nhau, instance nào cũng có thể xử lý yêu cầu đến. Việc phân bổ yêu cầu cho các instance đó gọi là *cân bằng tải* (*load balancing*) [^41]. Có nhiều giải pháp cân bằng tải và service discovery:

* *Bộ cân bằng tải phần cứng* (*hardware load balancer*) là thiết bị chuyên dụng được lắp trong trung tâm dữ liệu. Client chỉ kết nối tới một host và cổng, còn thiết bị định tuyến các kết nối đến tới một trong các server chạy dịch vụ đó. Loại cân bằng tải này phát hiện sự cố mạng khi kết nối tới server phía sau và chuyển lưu lượng sang server khác.
* *Bộ cân bằng tải phần mềm* (*software load balancer*) hoạt động gần giống bộ cân bằng tải phần cứng nhưng không cần thiết bị chuyên dụng. Các bộ cân bằng tải phần mềm như Nginx và HAProxy là ứng dụng có thể cài trên máy thông thường.
* *Hệ thống tên miền (DNS)* dùng để phân giải tên miền trên Internet, chẳng hạn khi bạn mở một trang Web. Nó cho phép một tên miền gắn với nhiều địa chỉ IP, từ đó thực hiện cân bằng tải. Client có thể được cấu hình để kết nối dịch vụ bằng tên miền thay vì địa chỉ IP, rồi tầng mạng của client chọn một địa chỉ IP khi thiết lập kết nối. Nhược điểm của cách này là DNS vốn được thiết kế để các thay đổi lan truyền qua một khoảng thời gian dài, và các bản ghi DNS bị cache. Nếu server khởi động, dừng hoặc di chuyển thường xuyên, client có thể nhận địa chỉ IP đã lỗi thời, nơi không còn server nào chạy.
* *Hệ thống service discovery* (*service discovery system*) không dùng DNS mà dùng một registry tập trung để theo dõi endpoint dịch vụ nào đang khả dụng. Khi một instance dịch vụ mới khởi động, nó đăng ký với hệ thống discovery, khai báo host và cổng nó lắng nghe cùng các metadata liên quan như thông tin shard sở hữu (xem [Chương 7](/ch7#ch_sharding)), vị trí trung tâm dữ liệu, v.v. Sau đó dịch vụ định kỳ gửi heartbeat tới hệ thống discovery để báo rằng nó vẫn khả dụng.

  Khi client muốn kết nối tới một dịch vụ, trước hết nó truy vấn hệ thống discovery để lấy danh sách endpoint khả dụng, rồi kết nối thẳng tới một endpoint. So với DNS, service discovery phù hợp hơn với môi trường động, nơi các instance thay đổi thường xuyên. Hệ thống discovery cũng cung cấp thêm metadata về dịch vụ cho client, giúp client đưa ra quyết định cân bằng tải sáng suốt hơn.
* *Service mesh* (*service mesh*) là một giải pháp cân bằng tải phức tạp hơn, kết hợp bộ cân bằng tải phần mềm với service discovery. Khác với bộ cân bằng tải phần mềm truyền thống chạy trên máy riêng, bộ cân bằng tải của service mesh thường được triển khai như thư viện client chạy trong tiến trình, hoặc như tiến trình hay container "sidecar" đi kèm client và server. Ứng dụng client kết nối tới bộ cân bằng tải dịch vụ cục bộ, bộ này kết nối tới bộ cân bằng tải phía server, rồi định tuyến kết nối tới tiến trình server trên máy cục bộ.

  Cấu trúc này tuy phức tạp nhưng có nhiều lợi thế. Vì cả ứng dụng client và server chỉ thiết lập kết nối cục bộ, việc mã hóa kết nối có thể do bộ cân bằng tải xử lý hoàn toàn, nên ứng dụng không phải đối mặt với chứng chỉ SSL và sự phức tạp của TLS. Service mesh còn cung cấp khả năng quan sát (observability) mạnh mẽ: theo dõi theo thời gian thực quan hệ gọi giữa các dịch vụ, phát hiện sự cố, giám sát tải lưu lượng, v.v.

Giải pháp nào phù hợp tùy vào nhu cầu của từng tổ chức. Trong các môi trường rất động dùng trình điều phối như Kubernetes, các tổ chức thường chọn service mesh như Istio hoặc Linkerd. Hạ tầng chuyên dụng như cơ sở dữ liệu hay hệ thống truyền thông điệp có thể cần bộ cân bằng tải đo ni đóng giày. Với các triển khai đơn giản hơn, bộ cân bằng tải phần mềm thường là đủ.

#### Mã hóa dữ liệu và tiến hóa cho RPC {#data-encoding-and-evolution-for-rpc}

Để đạt khả năng tiến hóa, client và server RPC phải có thể được thay đổi và triển khai độc lập. So với luồng dữ liệu qua cơ sở dữ liệu ở phần trước, với luồng dữ liệu qua dịch vụ ta có thể đưa ra một giả định đơn giản hóa: thường hợp lý khi cập nhật tất cả server trước, rồi mới cập nhật tất cả client. Do đó, ta chỉ cần tương thích ngược cho yêu cầu và tương thích xuôi cho phản hồi.

Tính chất tương thích ngược và xuôi của một giải pháp RPC phụ thuộc vào cách mã hóa mà nó dùng:

* gRPC (Protocol Buffers) và Avro RPC có thể tiến hóa theo các quy tắc tương thích của định dạng mã hóa tương ứng.
* API RESTful thường dùng JSON cho phản hồi, còn tham số yêu cầu thì thường dùng JSON, mã hóa URI hoặc mã hóa form. Thêm tham số yêu cầu tùy chọn, hoặc thêm trường mới vào đối tượng phản hồi, thường được coi là các thay đổi giữ được tương thích.

RPC thường được dùng để giao tiếp qua ranh giới giữa các tổ chức, nên khả năng tương thích của dịch vụ càng khó đảm bảo hơn: nhà cung cấp dịch vụ thường không kiểm soát được client và không thể ép họ nâng cấp. Do đó, tính tương thích cần được duy trì trong thời gian dài, thậm chí vô thời hạn. Nếu buộc phải thực hiện thay đổi phá vỡ tương thích, nhà cung cấp dịch vụ thường phải duy trì nhiều phiên bản của API dịch vụ cùng lúc.

Về việc nên đánh phiên bản API thế nào (tức làm sao client cho biết nó muốn dùng phiên bản API nào), chưa có sự đồng thuận chung [^42]. Với API RESTful, cách phổ biến là đưa số phiên bản vào URL hoặc vào header HTTP `Accept`. Nếu dịch vụ dùng khóa API để nhận diện từng client cụ thể, một lựa chọn khác là lưu phiên bản API mà client yêu cầu ở phía server và cho phép cập nhật lựa chọn phiên bản đó qua một giao diện quản trị riêng [^43].

### Durable execution và workflow {#sec_encoding_dataflow_workflows}

Theo định nghĩa, kiến trúc dựa trên dịch vụ gồm nhiều dịch vụ, mỗi dịch vụ phụ trách một phần của ứng dụng. Hãy lấy ứng dụng xử lý thanh toán làm ví dụ: nó cần trừ tiền thẻ tín dụng rồi gửi số tiền đó vào tài khoản ngân hàng. Rất có thể hệ thống dùng các dịch vụ riêng cho phát hiện gian lận, tích hợp thẻ tín dụng, tích hợp hệ thống ngân hàng, v.v.

Trong ví dụ này, xử lý một khoản thanh toán cần nhiều lời gọi dịch vụ. Dịch vụ xử lý thanh toán có thể gọi dịch vụ phát hiện gian lận để kiểm tra rủi ro, rồi gọi dịch vụ thẻ tín dụng để trừ tiền, rồi gọi dịch vụ ngân hàng để gửi khoản tiền đã trừ vào tài khoản, như trong {{< xref fig="5-7" page="/ch5" anchor="fig_encoding_workflow" >}}Hình 5-7{{< /xref >}}. Chuỗi bước này gọi là *workflow* (*workflow*), và mỗi bước gọi là một *tác vụ* (*task*). Workflow thường được định nghĩa như một đồ thị các tác vụ, bằng ngôn ngữ lập trình đa dụng, ngôn ngữ đặc thù miền (DSL), hoặc ngôn ngữ đánh dấu như Business Process Execution Language (BPEL) [^44].


> [!TIP] Tác vụ, activity và hàm
>
> Các workflow engine gọi tác vụ bằng những tên khác nhau. Chẳng hạn, Temporal dùng thuật ngữ *activity* (*activity*), còn một số engine khác gọi chúng là *durable function* (*durable function*). Tên khác nhau nhưng khái niệm giống nhau.


{{< fig num="5-7" id="fig_encoding_workflow" src="/fig/ddia_0507.png" caption="Ví dụ workflow biểu diễn bằng Business Process Model and Notation (BPMN), một ký pháp đồ họa." class="ddia-figure ddia-figure--panorama" width="2658" height="720" />}}


Workflow được chạy, hay thực thi, bởi một *workflow engine* (*workflow engine*). Engine quyết định mỗi tác vụ chạy khi nào, trên máy nào, phải làm gì khi tác vụ thất bại (chẳng hạn máy đang chạy tác vụ bị sập), cho phép bao nhiêu tác vụ chạy song song, v.v.

Workflow engine thường gồm một bộ điều phối (orchestrator) và một bộ thực thi (executor): bộ điều phối lên lịch, còn bộ thực thi thật sự chạy các tác vụ. Việc thực thi bắt đầu khi workflow được kích hoạt. Nếu người dùng định nghĩa lịch chạy theo thời gian, chẳng hạn mỗi giờ một lần, bộ điều phối có thể tự kích hoạt workflow. Một nguồn bên ngoài như dịch vụ Web, thậm chí một con người, cũng có thể kích hoạt workflow. Khi đã kích hoạt, các bộ thực thi được ra lệnh chạy tác vụ.

Có rất nhiều loại workflow engine phục vụ các trường hợp sử dụng khác nhau. Các engine như Airflow, Dagster và Prefect tích hợp với hệ thống dữ liệu và điều phối các tác vụ ETL. Các engine như Camunda và Orkes cung cấp biểu diễn workflow bằng đồ họa, như BPMN trong {{< xref fig="5-7" page="/ch5" anchor="fig_encoding_workflow" >}}Hình 5-7{{< /xref >}}, giúp người không phải kỹ sư dễ định nghĩa và thực thi workflow hơn. Các engine như Temporal và Restate cung cấp *durable execution* (*durable execution*).

#### Durable execution {#durable-execution}

Các framework durable execution đã trở thành cách phổ biến để xây dựng kiến trúc dịch vụ cần ngữ nghĩa giao dịch. Trong ví dụ thanh toán, ta muốn mỗi khoản thanh toán được xử lý đúng một lần. Nhưng sự cố trong lúc thực thi workflow có thể dẫn đến tình huống thẻ tín dụng đã bị trừ tiền mà tài khoản ngân hàng không nhận được khoản tiền tương ứng. Trong kiến trúc dựa trên dịch vụ, ta không thể đơn giản bọc hai tác vụ này vào một giao dịch cơ sở dữ liệu. Hơn nữa, hệ thống có thể còn phải tương tác với cổng thanh toán bên thứ ba mà ta không kiểm soát được đầy đủ.

Framework durable execution có thể cung cấp *ngữ nghĩa exactly-once* (*exactly-once semantics*) cho workflow. Nếu một tác vụ thất bại, framework thực thi lại nó, nhưng bỏ qua các lời gọi RPC hoặc thay đổi trạng thái đã hoàn tất thành công trước khi thất bại: framework có vẻ như gọi lại, nhưng thực chất trả ngay kết quả của lần gọi trước. Điều này khả thi vì framework ghi lại mọi RPC và thay đổi trạng thái vào một kho lưu trữ bền vững như write-ahead log (WAL) [^45] [^46]. {{< xref eg="5-5" page="/ch5" anchor="fig_temporal_workflow" >}}Ví dụ 5-5{{< /xref >}} cho thấy cách định nghĩa một workflow hỗ trợ durable execution bằng Temporal.

{{< eg num="5-5" id="fig_temporal_workflow" caption="Đoạn định nghĩa workflow bằng Temporal cho workflow thanh toán ở Hình 5-7" >}}
```python
@workflow.defn
class PaymentWorkflow:
    @workflow.run
    async def run(self, payment: PaymentRequest) -> PaymentResult:
        is_fraud = await workflow.execute_activity(
            check_fraud,
            payment,
            start_to_close_timeout=timedelta(seconds=15),
        )
        if is_fraud:
            return PaymentResultFraudulent
        credit_card_response = await workflow.execute_activity(
            debit_credit_card,
            payment,
            start_to_close_timeout=timedelta(seconds=15),
        )
        # ...
```
{{< /eg >}}

Các framework như Temporal không phải không có khó khăn. Các dịch vụ bên ngoài, chẳng hạn cổng thanh toán bên thứ ba trong ví dụ, vẫn phải cung cấp API idempotent, và nhà phát triển phải nhớ dùng ID duy nhất cho các lời gọi để tránh thực thi trùng lặp [^47]. Ngoài ra, vì framework durable execution ghi lại các lời gọi RPC theo thứ tự, nó đòi hỏi các lần thực thi sau phải gọi đúng như vậy theo cùng thứ tự. Điều này khiến thay đổi mã trở nên mong manh: chỉ cần đổi thứ tự các lời gọi hàm cũng có thể gây ra hành vi không xác định [^48]. Thay vì sửa mã của workflow hiện có, an toàn hơn là triển khai riêng một phiên bản mới, để các lần thực thi lại của workflow hiện có tiếp tục dùng mã cũ và chỉ workflow mới khởi động mới dùng mã mới [^49].

Tương tự, vì các framework durable execution yêu cầu phát lại toàn bộ mã theo cách tất định (cùng đầu vào phải cho cùng đầu ra), mã không tất định như bộ sinh số ngẫu nhiên hay đồng hồ hệ thống sẽ gây vấn đề [^48]. Framework thường cung cấp cài đặt tất định riêng cho các hàm thư viện kiểu này, nhưng nhà phát triển phải nhớ dùng chúng. Một số framework còn cung cấp công cụ phân tích tĩnh để kiểm tra xem có vô tình đưa vào hành vi không tất định hay không, như workflowcheck của Temporal.


> [!NOTE]
> Làm cho mã có tính tất định là một ý tưởng mạnh, nhưng khó thực hiện một cách đáng tin cậy. Ta sẽ quay lại chủ đề này ở [“Sức mạnh của tính tất định”](/ch9#sidebar_distributed_determinism).


### Kiến trúc hướng sự kiện {#sec_encoding_dataflow_msg}

Cuối cùng, ta điểm qua *kiến trúc hướng sự kiện* (*event-driven architecture*), một cách khác để dữ liệu đã mã hóa chảy giữa các tiến trình. Một yêu cầu ở đây được gọi là *sự kiện* (*event*) hoặc *message* (*message*). Khác với RPC, bên gửi thường không chờ bên nhận xử lý sự kiện. Hơn nữa, sự kiện thường không được gửi tới bên nhận qua kết nối mạng trực tiếp, mà qua một bên trung gian lưu tạm message, gọi là *message broker* (*message broker*), còn gọi là *event broker* (*event broker*), *message queue* (*message queue*) hoặc *middleware hướng message* (*message-oriented middleware*) [^50].

So với dùng RPC trực tiếp, message broker có vài ưu điểm:

* Nó có thể đóng vai trò bộ đệm khi bên nhận không khả dụng hoặc quá tải, nhờ đó cải thiện độ tin cậy của hệ thống.
* Nó có thể tự động gửi lại message cho tiến trình đã sập rồi phục hồi, nhờ đó tránh mất message.
* Nó không cần service discovery, vì bên gửi không phải kết nối trực tiếp tới địa chỉ IP của bên nhận.
* Nó cho phép gửi cùng một message tới nhiều bên nhận.
* Nó tách rời về mặt logic bên gửi và bên nhận: bên gửi chỉ phát hành message và không quan tâm ai tiêu thụ nó.

Giao tiếp qua message broker mang tính *bất đồng bộ* (*asynchronous*): bên gửi không chờ message được chuyển đến, mà chỉ gửi đi rồi quên nó. Tuy nhiên, bên gửi vẫn có thể chờ phản hồi trên một kênh riêng, từ đó đạt được mô hình giống RPC đồng bộ.

#### Message broker {#message-brokers}

Trước đây, lĩnh vực message broker chủ yếu do phần mềm doanh nghiệp thương mại của các công ty như TIBCO, IBM WebSphere và webMethods thống trị, sau đó các cài đặt mã nguồn mở như RabbitMQ, ActiveMQ, HornetQ, NATS và Apache Kafka dần trở nên phổ biến. Gần đây, các dịch vụ đám mây như Amazon Kinesis, Azure Service Bus và Google Cloud Pub/Sub cũng được áp dụng rộng rãi. Ta sẽ so sánh chúng chi tiết hơn ở [“Hệ thống message”](/ch12#sec_stream_messaging).

Ngữ nghĩa chuyển giao cụ thể khác nhau tùy cài đặt và cấu hình, nhưng phổ biến nhất là hai mẫu phân phối message sau:

* Một tiến trình thêm message vào một *hàng đợi* (*queue*) có tên, và broker chuyển message cho một *consumer* (*consumer*) của hàng đợi đó. Nếu có nhiều consumer, chỉ một trong số họ nhận message.
* Một tiến trình phát hành message lên một *topic* (*topic*) có tên, và broker chuyển message cho tất cả *subscriber* (*subscriber*) của topic đó. Nếu có nhiều subscriber, mỗi người đều nhận message.

Message broker thường không áp đặt mô hình dữ liệu cụ thể: message chỉ là dãy byte kèm một ít metadata, nên có thể dùng bất kỳ định dạng mã hóa nào. Cách làm phổ biến là dùng Protocol Buffers, Avro hoặc JSON, và triển khai một schema registry cạnh message broker để lưu mọi phiên bản schema hợp lệ và kiểm tra tính tương thích [^19] [^21]. AsyncAPI, đặc tả tương ứng với OpenAPI nhưng dành cho truyền message, cũng có thể dùng để chỉ định schema của message.

Mức độ bền vững của message khác nhau giữa các message broker. Nhiều broker ghi message ra đĩa để không mất khi broker sập hoặc khởi động lại. Tuy nhiên, khác với cơ sở dữ liệu, nhiều message broker tự động xóa message sau khi nó được tiêu thụ. Một số broker có thể cấu hình để lưu message vô thời hạn, điều này cần thiết nếu bạn muốn dùng event sourcing (xem [“Event sourcing và CQRS”](/ch3#sec_datamodels_events)).

Nếu consumer phát hành lại message lên một topic khác, bạn cần cẩn thận giữ nguyên các trường không biết, để tránh vấn đề đã bàn ở phần cơ sở dữ liệu ({{< xref fig="5-1" page="/ch5" anchor="fig_encoding_preserve_field" >}}Hình 5-1{{< /xref >}}).

#### Framework actor phân tán {#distributed-actor-frameworks}

*Mô hình actor* (*actor model*) là một mô hình lập trình cho tính đồng thời trong một tiến trình. Thay vì làm việc trực tiếp với luồng (thread) cùng các điều kiện tranh chấp, khóa và deadlock đi kèm, logic được đóng gói trong các *actor*. Mỗi actor thường đại diện cho một client hoặc thực thể, có thể có trạng thái cục bộ không chia sẻ với actor khác, và giao tiếp với các actor khác bằng cách gửi và nhận message bất đồng bộ. Việc truyền message không được đảm bảo: trong một số tình huống lỗi, message sẽ bị mất. Vì mỗi actor mỗi lần chỉ xử lý một message, nó không cần lo về luồng, và framework có thể lên lịch cho từng actor một cách độc lập.

Trong một *framework actor phân tán* (*distributed actor framework*) như Akka, Orleans [^51] và Erlang/OTP, mô hình lập trình này được dùng để mở rộng ứng dụng ra nhiều node. Cùng một cơ chế truyền message được dùng bất kể bên gửi và bên nhận ở cùng node hay khác node. Nếu ở khác node, message được mã hóa trong suốt thành dãy byte, gửi qua mạng và giải mã ở đầu bên kia.

Tính trong suốt về vị trí hoạt động tốt hơn trong mô hình actor so với trong RPC, vì mô hình actor vốn đã giả định message có thể bị mất, kể cả trong một tiến trình duy nhất. Dù độ trễ mạng có thể cao hơn độ trễ trong tiến trình, trong mô hình actor sự khác biệt căn bản giữa giao tiếp cục bộ và từ xa nhỏ hơn nhiều.

Một framework actor phân tán về bản chất tích hợp message broker và mô hình lập trình actor trong cùng một framework. Tuy nhiên, nếu bạn muốn rolling upgrade một ứng dụng dựa trên actor, vẫn phải cân nhắc tương thích xuôi và ngược: message có thể được gửi từ node chạy phiên bản mã mới tới node chạy phiên bản mã cũ, và ngược lại. Dùng một trong các cách mã hóa đã bàn trong chương này có thể đạt được khả năng tương thích đó.


## Tóm tắt {#summary}

Chương này đã xem xét một số cách chuyển cấu trúc dữ liệu thành byte trên mạng hoặc byte trên đĩa. Ta thấy rằng chi tiết của việc mã hóa không chỉ ảnh hưởng đến hiệu quả mà quan trọng hơn là cả kiến trúc của ứng dụng và các lựa chọn để triển khai, tiến hóa nó sau này.

Đặc biệt, nhiều dịch vụ cần hỗ trợ rolling upgrade, trong đó phiên bản mới của dịch vụ được triển khai dần lên một vài node thay vì lên tất cả cùng lúc. Rolling upgrade cho phép phát hành phiên bản mới của dịch vụ mà không gây ngừng hoạt động, và do đó khuyến khích phát hành các bản nhỏ thường xuyên thay vì các bản lớn hiếm hoi. Nó cũng giảm rủi ro triển khai, vì phiên bản lỗi được phát hiện và rút lại trước khi ảnh hưởng đến nhiều người dùng. Những tính chất này rất có lợi cho *khả năng tiến hóa*, tức mức độ dễ dàng khi thay đổi ứng dụng.

Trong rolling upgrade, hoặc vì nhiều lý do khác, ta phải giả định rằng các node khác nhau chạy các phiên bản mã ứng dụng khác nhau. Do đó, mọi dữ liệu chảy trong hệ thống đều nên được mã hóa theo cách vừa tương thích ngược (mã mới đọc được dữ liệu cũ) vừa tương thích xuôi (mã cũ đọc được dữ liệu mới).

Ta đã bàn nhiều định dạng mã hóa dữ liệu và tính chất tương thích của chúng:

* Các cách mã hóa riêng của ngôn ngữ lập trình chỉ dùng được với một ngôn ngữ và thường không cung cấp tương thích xuôi và ngược.
* Các định dạng văn bản như JSON, XML và CSV rất phổ biến, và tính tương thích của chúng phụ thuộc vào cách dùng. Chúng có các ngôn ngữ schema tùy chọn, đôi khi hữu ích và đôi khi lại là trở ngại. Các định dạng này có phần mơ hồ về kiểu dữ liệu, nên cần cẩn thận với những thứ như số và chuỗi nhị phân.
* Các định dạng nhị phân dựa trên schema như Protocol Buffers và Avro cho phép mã hóa gọn và hiệu quả với ngữ nghĩa tương thích xuôi và ngược được định nghĩa rõ ràng. Schema vừa có thể làm tài liệu, vừa có thể dùng để sinh mã cho các ngôn ngữ kiểu tĩnh. Tuy nhiên, chúng có nhược điểm là dữ liệu phải được giải mã trước khi con người đọc được.

Ta cũng đã bàn nhiều chế độ luồng dữ liệu, minh họa các tình huống mà việc mã hóa dữ liệu quan trọng:

* Trong cơ sở dữ liệu, tiến trình ghi vào cơ sở dữ liệu mã hóa dữ liệu, còn tiến trình đọc từ cơ sở dữ liệu giải mã nó.
* Trong RPC và REST API, client mã hóa yêu cầu, server giải mã yêu cầu và mã hóa phản hồi, rồi client giải mã phản hồi.
* Trong kiến trúc hướng sự kiện dùng message broker hoặc actor, các node giao tiếp bằng cách gửi message cho nhau, được bên gửi mã hóa và bên nhận giải mã.

Ta có thể kết luận rằng, chỉ cần chút cẩn thận, tương thích ngược, tương thích xuôi và rolling upgrade hoàn toàn khả thi. Chúc ứng dụng của bạn tiến hóa nhanh và được triển khai thường xuyên.


### Tài liệu tham khảo


[^1]: [CWE-502: Deserialization of Untrusted Data](https://cwe.mitre.org/data/definitions/502.html). Common Weakness Enumeration, *cwe.mitre.org*, July 2006. Archived at [perma.cc/26EU-UK9Y](https://perma.cc/26EU-UK9Y)
[^2]: Steve Breen. [What Do WebLogic, WebSphere, JBoss, Jenkins, OpenNMS, and Your Application Have in Common? This Vulnerability](https://foxglovesecurity.com/2015/11/06/what-do-weblogic-websphere-jboss-jenkins-opennms-and-your-application-have-in-common-this-vulnerability/). *foxglovesecurity.com*, November 2015. Archived at [perma.cc/9U97-UVVD](https://perma.cc/9U97-UVVD)
[^3]: Patrick McKenzie. [What the Rails Security Issue Means for Your Startup](https://www.kalzumeus.com/2013/01/31/what-the-rails-security-issue-means-for-your-startup/). *kalzumeus.com*, January 2013. Archived at [perma.cc/2MBJ-7PZ6](https://perma.cc/2MBJ-7PZ6)
[^4]: Brian Goetz. [Towards Better Serialization](https://openjdk.org/projects/amber/design-notes/towards-better-serialization). *openjdk.org*, June 2019. Archived at [perma.cc/UK6U-GQDE](https://perma.cc/UK6U-GQDE)
[^5]: Eishay Smith. [jvm-serializers wiki](https://github.com/eishay/jvm-serializers/wiki). *github.com*, October 2023. Archived at [perma.cc/PJP7-WCNG](https://perma.cc/PJP7-WCNG)
[^6]: [XML Is a Poor Copy of S-Expressions](https://wiki.c2.com/?XmlIsaPoorCopyOfEssExpressions). *wiki.c2.com*, May 2013. Archived at [perma.cc/7FAN-YBKL](https://perma.cc/7FAN-YBKL)
[^7]: Julia Evans. [Examples of floating point problems](https://jvns.ca/blog/2023/01/13/examples-of-floating-point-problems/). *jvns.ca*, January 2023. Archived at [perma.cc/M57L-QKKW](https://perma.cc/M57L-QKKW)
[^8]: Matt Harris. [Snowflake: An Update and Some Very Important Information](https://groups.google.com/g/twitter-development-talk/c/ahbvo3VTIYI). Email to *Twitter Development Talk* mailing list, October 2010. Archived at [perma.cc/8UBV-MZ3D](https://perma.cc/8UBV-MZ3D)
[^9]: Yakov Shafranovich. [RFC 4180: Common Format and MIME Type for Comma-Separated Values (CSV) Files](https://tools.ietf.org/html/rfc4180). IETF, October 2005.
[^10]: Andy Coates. [Evolving JSON Schemas - Part I](https://www.creekservice.org/articles/2024/01/08/json-schema-evolution-part-1.html) and [Part II](https://www.creekservice.org/articles/2024/01/09/json-schema-evolution-part-2.html). *creekservice.org*, January 2024. Archived at [perma.cc/MZW3-UA54](https://perma.cc/MZW3-UA54) and [perma.cc/GT5H-WKZ5](https://perma.cc/GT5H-WKZ5)
[^11]: Pierre Genevès, Nabil Layaïda, and Vincent Quint. [Ensuring Query Compatibility with Evolving XML Schemas](https://arxiv.org/abs/0811.4324). INRIA Technical Report 6711, November 2008.
[^12]: Tim Bray. [Bits On the Wire](https://www.tbray.org/ongoing/When/201x/2019/11/17/Bits-On-the-Wire). *tbray.org*, November 2019. Archived at [perma.cc/3BT3-BQU3](https://perma.cc/3BT3-BQU3)
[^13]: Mark Slee, Aditya Agarwal, and Marc Kwiatkowski. [Thrift: Scalable Cross-Language Services Implementation](https://thrift.apache.org/static/files/thrift-20070401.pdf). Facebook technical report, April 2007. Archived at [perma.cc/22BS-TUFB](https://perma.cc/22BS-TUFB)
[^14]: Martin Kleppmann. [Schema Evolution in Avro, Protocol Buffers and Thrift](https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html). *martin.kleppmann.com*, December 2012. Archived at [perma.cc/E4R2-9RJT](https://perma.cc/E4R2-9RJT)
[^15]: Doug Cutting, Chad Walters, Jim Kellerman, et al. [[PROPOSAL] New Subproject: Avro](https://lists.apache.org/thread/z571w0r5jmfsjvnl0fq4fgg0vh28d3bk). Email thread on *hadoop-general* mailing list, *lists.apache.org*, April 2009. Archived at [perma.cc/4A79-BMEB](https://perma.cc/4A79-BMEB)
[^16]: Apache Software Foundation. [Apache Avro 1.12.0 Specification](https://avro.apache.org/docs/1.12.0/specification/). *avro.apache.org*, August 2024. Archived at [perma.cc/C36P-5EBQ](https://perma.cc/C36P-5EBQ)
[^17]: Apache Software Foundation. [Avro schemas as LL(1) CFG definitions](https://avro.apache.org/docs/1.12.0/api/java/org/apache/avro/io/parsing/doc-files/parsing.html). *avro.apache.org*, August 2024. Archived at [perma.cc/JB44-EM9Q](https://perma.cc/JB44-EM9Q)
[^18]: Tony Hoare. [Null References: The Billion Dollar Mistake](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/). Talk at *QCon London*, March 2009.
[^19]: Confluent, Inc. [Schema Registry Overview](https://docs.confluent.io/platform/current/schema-registry/index.html). *docs.confluent.io*, 2024. Archived at [perma.cc/92C3-A9JA](https://perma.cc/92C3-A9JA)
[^20]: Aditya Auradkar and Tom Quiggle. [Introducing Espresso—LinkedIn’s Hot New Distributed Document Store](https://engineering.linkedin.com/espresso/introducing-espresso-linkedins-hot-new-distributed-document-store). *engineering.linkedin.com*, January 2015. Archived at [perma.cc/FX4P-VW9T](https://perma.cc/FX4P-VW9T)
[^21]: Jay Kreps. [Putting Apache Kafka to Use: A Practical Guide to Building a Stream Data Platform (Part 2)](https://www.confluent.io/blog/event-streaming-platform-2/). *confluent.io*, February 2015. Archived at [perma.cc/8UA4-ZS5S](https://perma.cc/8UA4-ZS5S)
[^22]: Gwen Shapira. [The Problem of Managing Schemas](https://www.oreilly.com/content/the-problem-of-managing-schemas/). *oreilly.com*, November 2014. Archived at [perma.cc/BY8Q-RYV3](https://perma.cc/BY8Q-RYV3)
[^23]: John Larmouth. [*ASN.1 Complete*](https://www.oss.com/asn1/resources/books-whitepapers-pubs/larmouth-asn1-book.pdf). Morgan Kaufmann, 1999. ISBN: 978-0-122-33435-1. Archived at [perma.cc/GB7Y-XSXQ](https://perma.cc/GB7Y-XSXQ)
[^24]: Burton S. Kaliski Jr. [A Layman’s Guide to a Subset of ASN.1, BER, and DER](https://luca.ntop.org/Teaching/Appunti/asn1.html). Technical Note, RSA Data Security, Inc., November 1993. Archived at [perma.cc/2LMN-W9U8](https://perma.cc/2LMN-W9U8)
[^25]: Jacob Hoffman-Andrews. [A Warm Welcome to ASN.1 and DER](https://letsencrypt.org/docs/a-warm-welcome-to-asn1-and-der/). *letsencrypt.org*, April 2020. Archived at [perma.cc/CYT2-GPQ8](https://perma.cc/CYT2-GPQ8)
[^26]: Lev Walkin. [Question: Extensibility and Dropping Fields](https://lionet.info/asn1c/blog/2010/09/21/question-extensibility-removing-fields/). *lionet.info*, September 2010. Archived at [perma.cc/VX8E-NLH3](https://perma.cc/VX8E-NLH3)
[^27]: Jacqueline Xu. [Online migrations at scale](https://stripe.com/blog/online-migrations). *stripe.com*, February 2017. Archived at [perma.cc/X59W-DK7Y](https://perma.cc/X59W-DK7Y)
[^28]: Geoffrey Litt, Peter van Hardenberg, and Orion Henry. [Project Cambria: Translate your data with lenses](https://www.inkandswitch.com/cambria/). Technical Report, *Ink & Switch*, October 2020. Archived at [perma.cc/WA4V-VKDB](https://perma.cc/WA4V-VKDB)
[^29]: Pat Helland. [Data on the Outside Versus Data on the Inside](https://www.cidrdb.org/cidr2005/papers/P12.pdf). At *2nd Biennial Conference on Innovative Data Systems Research* (CIDR), January 2005.
[^30]: Roy Thomas Fielding. [Architectural Styles and the Design of Network-Based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf). PhD Thesis, University of California, Irvine, 2000. Archived at [perma.cc/LWY9-7BPE](https://perma.cc/LWY9-7BPE)
[^31]: Roy Thomas Fielding. [REST APIs must be hypertext-driven](https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven).” *roy.gbiv.com*, October 2008. Archived at [perma.cc/M2ZW-8ATG](https://perma.cc/M2ZW-8ATG)
[^32]: [OpenAPI Specification Version 3.1.0](https://swagger.io/specification/). *swagger.io*, February 2021. Archived at [perma.cc/3S6S-K5M4](https://perma.cc/3S6S-K5M4)
[^33]: Michi Henning. [The Rise and Fall of CORBA](https://cacm.acm.org/practice/the-rise-and-fall-of-corba/). *Communications of the ACM*, volume 51, issue 8, pages 52–57, August 2008. [doi:10.1145/1378704.1378718](https://doi.org/10.1145/1378704.1378718)
[^34]: Pete Lacey. [The S Stands for Simple](https://harmful.cat-v.org/software/xml/soap/simple). *harmful.cat-v.org*, November 2006. Archived at [perma.cc/4PMK-Z9X7](https://perma.cc/4PMK-Z9X7)
[^35]: Stefan Tilkov. [Interview: Pete Lacey Criticizes Web Services](https://www.infoq.com/articles/pete-lacey-ws-criticism/). *infoq.com*, December 2006. Archived at [perma.cc/JWF4-XY3P](https://perma.cc/JWF4-XY3P)
[^36]: Tim Bray. [The Loyal WS-Opposition](https://www.tbray.org/ongoing/When/200x/2004/09/18/WS-Oppo). *tbray.org*, September 2004. Archived at [perma.cc/J5Q8-69Q2](https://perma.cc/J5Q8-69Q2)
[^37]: Andrew D. Birrell and Bruce Jay Nelson. [Implementing Remote Procedure Calls](https://www.cs.princeton.edu/courses/archive/fall03/cs518/papers/rpc.pdf). *ACM Transactions on Computer Systems* (TOCS), volume 2, issue 1, pages 39–59, February 1984. [doi:10.1145/2080.357392](https://doi.org/10.1145/2080.357392)
[^38]: Jim Waldo, Geoff Wyant, Ann Wollrath, and Sam Kendall. [A Note on Distributed Computing](https://m.mirror.facebook.net/kde/devel/smli_tr-94-29.pdf). Sun Microsystems Laboratories, Inc., Technical Report TR-94-29, November 1994. Archived at [perma.cc/8LRZ-BSZR](https://perma.cc/8LRZ-BSZR)
[^39]: Steve Vinoski. [Convenience over Correctness](https://steve.vinoski.net/pdf/IEEE-Convenience_Over_Correctness.pdf). *IEEE Internet Computing*, volume 12, issue 4, pages 89–92, July 2008. [doi:10.1109/MIC.2008.75](https://doi.org/10.1109/MIC.2008.75)
[^40]: Brandur Leach. [Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency). *stripe.com*, February 2017. Archived at [perma.cc/JD22-XZQT](https://perma.cc/JD22-XZQT)
[^41]: Sam Rose. [Load Balancing](https://samwho.dev/load-balancing/). *samwho.dev*, April 2023. Archived at [perma.cc/Q7BA-9AE2](https://perma.cc/Q7BA-9AE2)
[^42]: Troy Hunt. [Your API versioning is wrong, which is why I decided to do it 3 different wrong ways](https://www.troyhunt.com/your-api-versioning-is-wrong-which-is/). *troyhunt.com*, February 2014. Archived at [perma.cc/9DSW-DGR5](https://perma.cc/9DSW-DGR5)
[^43]: Brandur Leach. [APIs as infrastructure: future-proofing Stripe with versioning](https://stripe.com/blog/api-versioning). *stripe.com*, August 2017. Archived at [perma.cc/L63K-USFW](https://perma.cc/L63K-USFW)
[^44]: Alexandre Alves, Assaf Arkin, Sid Askary, et al. [Web Services Business Process Execution Language Version 2.0](https://docs.oasis-open.org/wsbpel/2.0/wsbpel-v2.0.html). *docs.oasis-open.org*, April 2007.
[^45]: [What is a Temporal Service?](https://docs.temporal.io/clusters) *docs.temporal.io*, 2024. Archived at [perma.cc/32P3-CJ9V](https://perma.cc/32P3-CJ9V)
[^46]: Stephan Ewen. [Why we built Restate](https://restate.dev/blog/why-we-built-restate/). *restate.dev*, August 2023. Archived at [perma.cc/BJJ2-X75K](https://perma.cc/BJJ2-X75K)
[^47]: Keith Tenzer and Joshua Smith. [Idempotency and Durable Execution](https://temporal.io/blog/idempotency-and-durable-execution). *temporal.io*, February 2024. Archived at [perma.cc/9LGW-PCLU](https://perma.cc/9LGW-PCLU)
[^48]: [What is a Temporal Workflow?](https://docs.temporal.io/workflows) *docs.temporal.io*, 2024. Archived at [perma.cc/B5C5-Y396](https://perma.cc/B5C5-Y396)
[^49]: Jack Kleeman. [Solving durable execution’s immutability problem](https://restate.dev/blog/solving-durable-executions-immutability-problem/). *restate.dev*, February 2024. Archived at [perma.cc/G55L-EYH5](https://perma.cc/G55L-EYH5)
[^50]: Srinath Perera. [Exploring Event-Driven Architecture: A Beginner’s Guide for Cloud Native Developers](https://wso2.com/blogs/thesource/exploring-event-driven-architecture-a-beginners-guide-for-cloud-native-developers/). *wso2.com*, August 2023. Archived at [archive.org](https://web.archive.org/web/20240716204613/https%3A//wso2.com/blogs/thesource/exploring-event-driven-architecture-a-beginners-guide-for-cloud-native-developers/)
[^51]: Philip A. Bernstein, Sergey Bykov, Alan Geller, Gabriel Kliot, and Jorgen Thelin. [Orleans: Distributed Virtual Actors for Programmability and Scalability](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/). Microsoft Research Technical Report MSR-TR-2014-41, March 2014. Archived at [perma.cc/PD3U-WDMF](https://perma.cc/PD3U-WDMF)
