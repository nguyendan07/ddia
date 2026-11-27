---
translationKey: ddia-v2-preface
title: Lời nói đầu
book_kind: preface
weight: 50
breadcrumbs: false
---

> [!NOTE]
> Trang này giữ nguyên bản dịch lời nói đầu của ấn bản thứ nhất; phần "Tổng quan về sách" do trang này duy trì theo mục lục ấn bản thứ hai.

Nếu mấy năm gần đây bạn làm kỹ thuật phần mềm, nhất là phát triển hệ thống phía máy chủ và backend, rất có thể bạn đã bị "oanh tạc" bởi hàng loạt từ thời thượng về lưu trữ và xử lý dữ liệu: NoSQL! Big Data! Web-Scale! Sharding! Nhất quán cuối cùng! ACID! Định lý CAP! Dịch vụ đám mây! MapReduce! Thời gian thực!

Trong mười năm qua, chúng ta đã chứng kiến nhiều tiến bộ thú vị về cơ sở dữ liệu, hệ thống phân tán và cách xây dựng ứng dụng trên nền chúng. Những tiến bộ này có nhiều động lực khác nhau:

* Các công ty internet như Google, Yahoo!, Amazon, Facebook, LinkedIn, Microsoft và Twitter đang xử lý lượng dữ liệu và lưu lượng khổng lồ, buộc họ phải tạo ra những công cụ mới để đối phó hiệu quả với quy mô đó.
* Doanh nghiệp cần linh hoạt, kiểm chứng giả thuyết với chi phí thấp, và phản ứng nhanh trước những nhận định mới về thị trường bằng cách rút ngắn chu kỳ phát triển và giữ cho mô hình dữ liệu linh hoạt.
* Phần mềm tự do và mã nguồn mở đã rất thành công, và trong nhiều môi trường được ưa chuộng hơn phần mềm thương mại hay phần mềm tự phát triển nội bộ.
* Tốc độ xung nhịp CPU gần như không tăng, nhưng bộ xử lý đa nhân đã trở thành tiêu chuẩn và mạng ngày càng nhanh. Điều này có nghĩa là mức độ song song hóa chỉ tăng chứ không giảm.
* Ngay cả khi làm việc trong một nhóm nhỏ, giờ đây bạn cũng có thể xây dựng hệ thống phân bổ trên nhiều máy, thậm chí nhiều vùng địa lý, nhờ những đơn vị tiên phong trong khái niệm hạ tầng như một dịch vụ (IaaS) như Amazon Web Services (AWS).
* Nhiều dịch vụ ngày nay được kỳ vọng có tính sẵn sàng cao; việc ngừng hoạt động kéo dài do mất điện hay bảo trì ngày càng khó chấp nhận.

Các *ứng dụng chuyên sâu về dữ liệu* (*data-intensive applications*) đang đẩy lùi ranh giới của những điều khả dĩ bằng cách tận dụng các tiến bộ công nghệ này. Một ứng dụng được gọi là *chuyên sâu về dữ liệu* nếu *dữ liệu là thách thức chính của nó* (khối lượng dữ liệu, độ phức tạp của dữ liệu hoặc tốc độ thay đổi của dữ liệu) — trái với *chuyên sâu về tính toán* (*compute-intensive*), nơi tốc độ bộ xử lý là điểm nghẽn.

Các công cụ và kỹ thuật giúp ứng dụng chuyên sâu về dữ liệu lưu trữ và xử lý dữ liệu đang thích ứng nhanh chóng với những thay đổi này. Các loại hệ quản trị cơ sở dữ liệu mới ("NoSQL") đã thu hút nhiều sự chú ý, còn hàng đợi thông điệp, cache, chỉ mục tìm kiếm, các framework xử lý theo lô và xử lý luồng cùng những công nghệ liên quan cũng rất quan trọng. Nhiều ứng dụng kết hợp những công cụ và kỹ thuật này với nhau.

Những từ thời thượng tràn ngập lĩnh vực này cho thấy sự hào hứng với các khả năng mới, và đó là điều tốt. Tuy nhiên, là kỹ sư phần mềm và kiến trúc sư, để xây dựng được những ứng dụng tốt, chúng ta còn cần hiểu biết kỹ thuật chính xác về vô số công nghệ liên tục xuất hiện cùng những đánh đổi của chúng. Để có được cái nhìn thấu đáo đó, ta cần đào sâu vào những gì nằm sau các từ thời thượng.

May mắn là, đằng sau những thay đổi công nghệ nhanh chóng luôn có những nguyên lý bền vững, vẫn đúng bất kể bạn dùng phiên bản nào của một công cụ cụ thể. Nếu hiểu những nguyên lý này, bạn có thể nắm được từng công cụ phù hợp với tình huống nào, cách khai thác chúng tốt nhất và cách tránh những cạm bẫy. Đó chính là mục đích của cuốn sách này.

Mục tiêu của cuốn sách là giúp bạn định hướng trong bức tranh đa dạng và biến đổi nhanh của các công nghệ xử lý và lưu trữ dữ liệu. Đây không phải hướng dẫn cho một công cụ cụ thể nào, cũng không phải một cuốn giáo trình đầy lý thuyết khô khan. Thay vào đó, ta sẽ xem một số ví dụ về những hệ thống dữ liệu thành công: các kỹ thuật này là nền tảng của nhiều ứng dụng phổ biến, vốn hằng ngày phải đáp ứng yêu cầu về khả năng mở rộng, hiệu năng và độ tin cậy trong môi trường production.

Ta sẽ đi sâu vào bên trong các hệ thống đó, gỡ rối các thuật toán then chốt, thảo luận các nguyên lý đằng sau và những đánh đổi mà chúng buộc phải thực hiện. Trong hành trình này, ta sẽ cố gắng tìm những cách hiệu quả để *suy nghĩ* về hệ thống dữ liệu — không chỉ về việc chúng hoạt động *như thế nào*, mà còn *vì sao* chúng hoạt động theo cách đó, và những câu hỏi nào ta cần đặt ra.

Đọc xong cuốn sách này, bạn sẽ có khả năng quyết định công nghệ nào phù hợp với mục đích nào, và hiểu cách kết hợp các công cụ để làm nền tảng cho một kiến trúc ứng dụng tốt. Cuốn sách chưa đủ để bạn tự xây dựng một storage engine cơ sở dữ liệu từ đầu, nhưng may mắn là điều đó hầu như hiếm khi cần thiết. Bạn sẽ có trực giác nhạy bén về những gì diễn ra bên dưới hệ thống, nhờ đó đủ khả năng suy luận về hành vi của chúng, đưa ra các quyết định thiết kế tốt và lần ra mọi vấn đề có thể phát sinh.


## Đối tượng độc giả

Nếu bạn phát triển ứng dụng có một dạng máy chủ / hệ thống backend nào đó để lưu trữ hoặc xử lý dữ liệu, và ứng dụng dùng mạng (ví dụ ứng dụng web, ứng dụng di động hoặc cảm biến kết nối internet), thì cuốn sách này dành cho bạn.

Sách dành cho kỹ sư phần mềm, kiến trúc sư phần mềm và các nhà quản lý kỹ thuật yêu thích việc viết mã. Sách đặc biệt hữu ích nếu bạn cần đưa ra quyết định về kiến trúc của hệ thống mình đang làm — ví dụ chọn công cụ giải quyết một vấn đề cụ thể và tìm cách dùng chúng tốt nhất. Nhưng ngay cả khi bạn không có quyền chọn công cụ, sách vẫn giúp bạn hiểu rõ hơn điểm mạnh và điểm yếu của những công cụ đang dùng.

Bạn nên có một số kinh nghiệm xây dựng ứng dụng web hoặc dịch vụ mạng, và nên quen thuộc với cơ sở dữ liệu quan hệ và SQL. Hiểu biết về bất kỳ cơ sở dữ liệu phi quan hệ hay công cụ nào khác liên quan đến dữ liệu sẽ có ích, nhưng không bắt buộc. Có hiểu biết chung về các giao thức mạng phổ biến như TCP và HTTP cũng hữu ích. Việc chọn ngôn ngữ lập trình hay framework không ảnh hưởng gì đến việc đọc sách này.

Nếu bất kỳ điều nào sau đây đúng với bạn, bạn sẽ thấy cuốn sách này có giá trị:

* Bạn muốn tìm hiểu cách làm cho hệ thống dữ liệu có khả năng mở rộng, ví dụ để hỗ trợ ứng dụng web hoặc di động có hàng triệu người dùng.
* Bạn cần nâng cao tính sẵn sàng của ứng dụng (giảm thiểu thời gian ngừng hoạt động) và giữ cho nó vận hành ổn định.
* Bạn đang tìm cách làm cho hệ thống dễ bảo trì về lâu dài, ngay cả khi hệ thống lớn lên, và yêu cầu cùng công nghệ thay đổi.
* Bạn có sự tò mò tự nhiên về cách mọi thứ vận hành, và muốn biết điều gì diễn ra phía sau các trang web và dịch vụ trực tuyến lớn. Cuốn sách này mổ xẻ phần bên trong của nhiều cơ sở dữ liệu và hệ thống xử lý dữ liệu, và khám phá trí tuệ trong thiết kế của chúng là điều rất thú vị.

Đôi khi, khi bàn về hệ thống dữ liệu có khả năng mở rộng, người ta nói: "Bạn đâu phải Google hay Amazon, đừng lo về khả năng mở rộng, cứ dùng cơ sở dữ liệu quan hệ đi." Nhận định này có phần đúng: thiết kế chương trình cho khả năng mở rộng không cần thiết vừa tốn công vô ích, vừa có thể khóa bạn vào một thiết kế thiếu linh hoạt. Thực ra đây là một dạng "tối ưu hóa sớm". Tuy nhiên, chọn đúng công cụ quả thật quan trọng, và mỗi công nghệ có ưu nhược điểm riêng. Ta sẽ thấy rằng cơ sở dữ liệu quan hệ tuy quan trọng nhưng tuyệt không phải điểm cuối của việc xử lý dữ liệu.


## Phạm vi của sách

Cuốn sách này không cố hướng dẫn người đọc cách cài đặt hay dùng một gói phần mềm hoặc API cụ thể, vì đã có rất nhiều tài liệu đưa ra hướng dẫn chi tiết. Thay vào đó, ta bàn về nền tảng của hệ thống dữ liệu — các nguyên lý và đánh đổi — và khám phá những quyết định thiết kế khác nhau của từng sản phẩm.

Bản sách điện tử có kèm liên kết đến toàn văn các nguồn trực tuyến. Mọi liên kết đều được kiểm tra vào thời điểm xuất bản, nhưng tiếc là do quy luật tự nhiên của web, liên kết thường xuyên bị hỏng. Nếu gặp liên kết hỏng, hoặc đang đọc bản in, bạn có thể dùng công cụ tìm kiếm để tìm tài liệu tham khảo. Với các bài báo học thuật, bạn có thể tìm tiêu đề trên Google Scholar để lấy bản PDF công khai. Hoặc bạn có thể tìm toàn bộ tài liệu tham khảo tại https://github.com/ept/ddia-references, nơi chúng tôi duy trì các liên kết mới nhất.

Trọng tâm của chúng ta là *kiến trúc* (*architecture*) của các hệ thống dữ liệu và cách chúng được tích hợp vào các ứng dụng chuyên sâu về dữ liệu. Cuốn sách không đủ chỗ để bao quát các lĩnh vực triển khai, vận hành, bảo mật, quản trị — đây đều là những chủ đề phức tạp và quan trọng, nên chỉ bàn qua loa vài ghi chú trong sách này là không công bằng với chúng. Mỗi lĩnh vực xứng đáng có một cuốn sách riêng.

Nhiều kỹ thuật được mô tả trong sách này nằm trong phạm trù của từ thời thượng *Big Data*. Tuy nhiên, thuật ngữ "big data" bị lạm dụng và thiếu định nghĩa rõ ràng đến mức vô dụng trong các thảo luận kỹ thuật nghiêm túc. Cuốn sách này dùng những thuật ngữ ít mơ hồ hơn, chẳng hạn "một node" đối lập với "hệ thống phân tán", hoặc "hệ thống trực tuyến / tương tác" đối lập với "hệ thống ngoại tuyến / xử lý theo lô".

Sách có phần ưu ái *phần mềm tự do và mã nguồn mở* (*FOSS*), vì đọc, sửa và chạy mã nguồn là cách tốt để tìm hiểu chi tiết cách một thứ hoạt động. Các nền tảng mở cũng giảm rủi ro bị nhà cung cấp độc quyền chi phối. Tuy nhiên, khi phù hợp, ta cũng bàn về phần mềm sở hữu độc quyền (phần mềm mã nguồn đóng, phần mềm như một dịch vụ SaaS, hoặc một số phần mềm nội bộ của công ty đã được mô tả trong tài liệu nhưng chưa phát hành công khai).

## Tổng quan về sách

Dưới đây là mục lục các chương của ấn bản thứ hai do trang này duy trì:

1. [Phần I: Nền tảng của hệ thống dữ liệu](/part-i): [Chương 1: Đánh đổi trong kiến trúc hệ thống dữ liệu](/ch1), [Chương 2: Định nghĩa yêu cầu phi chức năng](/ch2), [Chương 3: Mô hình dữ liệu và ngôn ngữ truy vấn](/ch3), [Chương 4: Lưu trữ và truy xuất](/ch4), [Chương 5: Mã hóa và tiến hóa](/ch5).

2. [Phần II: Dữ liệu phân tán](/part-ii): [Chương 6: Sao chép](/ch6), [Chương 7: Phân mảnh](/ch7), [Chương 8: Giao dịch](/ch8), [Chương 9: Những rắc rối của hệ phân tán](/ch9), [Chương 10: Nhất quán và đồng thuận](/ch10).

3. [Phần III: Dữ liệu dẫn xuất](/part-iii): [Chương 11: Xử lý theo lô](/ch11), [Chương 12: Xử lý luồng](/ch12), [Chương 13: Triết lý của hệ thống xử lý luồng](/ch13), [Chương 14: Làm điều đúng đắn](/ch14).


## Tài liệu tham khảo và đọc thêm

Phần lớn nội dung bàn trong sách này đã xuất hiện ở đâu đó dưới một hình thức nào đó — bài thuyết trình hội nghị, bài báo nghiên cứu, bài blog, mã nguồn, hệ thống theo dõi lỗi, danh sách thư và thực hành kỹ thuật. Cuốn sách tổng hợp những ý tưởng quan trọng nhất từ các nguồn khác nhau, và trong văn bản có kèm liên kết đến tài liệu gốc. Nếu muốn khám phá sâu hơn một lĩnh vực, phần tài liệu tham khảo ở cuối mỗi chương là nguồn rất tốt, phần lớn có thể truy cập miễn phí trực tuyến.


## O‘Reilly Safari

[Safari](http://oreilly.com/safari) (formerly Safari Books Online) is a membership-based training and reference platform for enterprise, government, educators, and individuals.

Members have access to thousands of books, training videos, Learning Paths, interac‐ tive tutorials, and curated playlists from over 250 publishers, including O’Reilly Media, Harvard Business Review, Prentice Hall Professional, Addison-Wesley Pro‐ fessional, Microsoft Press, Sams, Que, Peachpit Press, Adobe, Focal Press, Cisco Press, John Wiley & Sons, Syngress, Morgan Kaufmann, IBM Redbooks, Packt, Adobe Press, FT Press, Apress, Manning, New Riders, McGraw-Hill, Jones & Bartlett, and Course Technology, among others.

For more information, please visit http://oreilly.com/safari.


## Liên hệ với chúng tôi

Mọi bình luận và câu hỏi về cuốn sách xin gửi đến nhà xuất bản:

O’Reilly Media, Inc.  
1005 Gravenstein Highway North  
Sebastopol, CA 95472  
800-998-9938 (Hoa Kỳ hoặc Canada)  
707-829-0515 (quốc tế hoặc nội hạt)  
707-829-0104 (fax)

Chúng tôi có một trang web cho cuốn sách, nơi liệt kê đính chính, ví dụ và mọi thông tin bổ sung. Bạn có thể truy cập: *http://bit.ly/designing-data-intensive-apps*.

Để bình luận hoặc đặt câu hỏi kỹ thuật, vui lòng gửi email đến: *bookquestions@oreilly.com*.

Để biết thêm thông tin về sách, khóa học, hội nghị và tin tức của O’Reilly, hãy truy cập: *http://www.oreilly.com*.

* Facebook: [http://facebook.com/oreilly](http://facebook.com/oreilly)
* Twitter: [http://twitter.com/oreillymedia](http://twitter.com/oreillymedia)
* YouTube: [http://www.youtube.com/oreillymedia](http://www.youtube.com/oreillymedia)


## Lời cảm ơn

Cuốn sách này kết hợp kinh nghiệm từ nghiên cứu học thuật lẫn thực tiễn công nghiệp, tổng hợp và hệ thống hóa vô số ý tưởng và tri thức của người khác. Trong ngành máy tính, ta thường bị cuốn hút bởi những thứ mới lạ, nhưng tôi cho rằng công việc của những người đi trước còn rất nhiều điều đáng học. Cuốn sách có hơn 800 trích dẫn: bài viết, blog, bài giảng, tài liệu… — đối với tôi, đó đều là nguồn học tập quý giá. Tôi rất biết ơn các tác giả đã chia sẻ tri thức của mình.

Tôi cũng học được rất nhiều từ những cuộc trò chuyện, khi nhiều người đã dành thời gian quý báu để thảo luận ý tưởng và kiên nhẫn giải thích với tôi. Đặc biệt cảm ơn Joe Adler, Ross Anderson, Peter Bailis, Márton Balassi, Alastair Beresford, Mark Callaghan, Mat Clayton, Patrick Collison, Sean Cribbs, Shirshanka Das, Niklas Ekström, Stephan Ewen, Alan Fekete, Gyula Fóra, Camille Fournier, Andres Freund, John Garbutt, Seth Gilbert, Tom Haggett, Pat Hel‐ land, Joe Hellerstein, Jakob Homan, Heidi Howard, John Hugg, Julian Hyde, Conrad Irwin, Evan Jones, Flavio Junqueira, Jessica Kerr, Kyle Kingsbury, Jay Kreps, Carl Lerche, Nicolas Liochon, Steve Loughran, Lee Mallabone, Nathan Marz, Caitie McCaffrey, Josie McLellan, Christopher Meiklejohn, Ian Meyers, Neha Narkhede, Neha Narula, Cathy O’Neil, Onora O’Neill, Ludovic Orban, Zoran Perkov, Julia Powles, Chris Riccomini, Henry Robinson, David Rosenthal, Jennifer Rullmann, Matthew Sackman, Martin Scholl, Amit Sela, Gwen Shapira, Greg Spurrier, Sam Stokes, Ben Stopford, Tom Stuart, Diana Vasile, Rahul Vohra, Pete Warden, và Brett Wooldridge.

Nhiều người khác đã đóng góp vô giá trong quá trình viết sách bằng cách đọc duyệt bản thảo và đưa ra ý kiến phản hồi. Tôi đặc biệt cảm ơn Raul Agepati, Tyler Akidau, Mattias Andersson, Sasha Baranov, Veena Basavaraj, David Beyer, Jim Brikman, Paul Carey, Raul Castro Fernandez, Joseph Chow, Derek Elkins, Sam Elliott, Alexander Gallego, Mark Grover, Stu Halloway, Heidi Howard, Nicola Kleppmann, Stefan Kruppa, Bjorn Madsen, Sander Mak, Stefan Podkowinski, Phil Potter, Hamid Ramazani, Sam Stokes, và Ben Summers. Dĩ nhiên, mọi sai sót còn sót lại hay quan điểm khó chấp nhận trong sách đều do tôi hoàn toàn chịu trách nhiệm.

Để cuốn sách có thể ra đời, và kiên nhẫn đối phó với tốc độ viết chậm cùng những yêu cầu khác thường của tôi, tôi xin cảm ơn các biên tập viên Marie Beaugureau, Mike Loukides, Ann Spencer và toàn bộ đội ngũ O'Reilly. Tôi xin cảm ơn Rachel Head đã giúp tôi tìm được thuật ngữ phù hợp. Tôi xin cảm ơn Alastair Beresford, Susan Goodhue, Neha Narkhede và Kevin Scott đã dành cho tôi thời gian và sự tự do để viết, bên ngoài các công việc khác.

Đặc biệt cảm ơn Shabbir Diwan và Edie Freedman, những người đã dày công vẽ bản đồ cho từng chương. Họ đưa ra những ý tưởng không theo lối mòn và tạo ra các bản đồ đẹp, cuốn hút, thật tuyệt vời.

Cuối cùng, tôi gửi lời yêu thương đến gia đình và bạn bè, nếu không có họ, tôi đã không thể đi hết hành trình viết sách gần bốn năm này. Các bạn là tuyệt nhất.
