---
title: 'System Design: Thiết kế một hệ thống tạo ID'
date: 2023-05-04 00:53:00 +0700
categories: [Blogging]
tags: [System Design]
pin: false
---
## Dẫn nhập

Bài toán: Trong một hệ thống phân tán nhiều server, cần thiết kế một hệ thống nhằm khởi tạo ID hiệu quả

Phân tích: hiện tại trong các cơ sở dữ liệu có hỗ trợ một số cách để sinh ra ID. Có thể kể đến 2 cách phổ biến như: Số tự nhiên tự tăng (auto-increment integer) và UUID/GUID

Nói về kiểu số tự nhiên tự tăng, cách này có đặc điểm là số nên có thể sắp xếp, dữ liệu được tạo ra trễ hơn thì có id lớn hơn. Tuy nhiên không có tính duy nhất khi xét trong toàn hệ thống phân tán. 

Có một giải pháp được sử dụng để khắc phục cho điểm yếu của số tự nhiên tự tăng, đó chính là sử dụng một server chuyên sinh ra ID tăng dần cho toàn bộ hệ thống. Việc này sẽ đảm bảo ID là duy nhất ở toàn bộ hệ thống và còn có thể sắp xếp được. Cách này được sử dụng bởi Flickr (link bài viết ở đây: [link](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)) Nhược điểm của cách này là khi request quá lớn, server này có thể là một điểm nghẽn cho toàn bộ hệ thống (bottelneck)

Ngược lại, giá trị UUID/GUID được sinh ra lại có tính duy nhất trên toàn hệ thống, nhưng lại không có khả năng sắp xếp, hơn nữa lại chiếm 128 bit cho mỗi giá trị, nhiều hơn so với chỉ 64 bit trong các cơ sở dữ liệu phổ biến

Như vậy, điều kiện của một hệ thống sinh ID hiệu quả có thể kể đến như sau:

- Quan trọng nhất: Phải duy nhất trên toàn bộ hệ thống
- Có thể sắp xếp: vì các dữ liệu được lưu trữ phân tán ở các server khác nhau, việc có thể sắp xếp còn thuận lợi cho việc tìm kiếm
- Không chiếm quá nhiều dung lượng
- Dễ tạo: Thuật toán khởi tạo phải đơn giản, không mất nhiều thời gian mỗi khi cần khởi tạo để tránh quá tải khi nhu cầu tạo ID tăng đột biến
- Dễ scale: khi cần nâng số server trong một hệ thống phân tán thì thuật toán khởi tạo có thể dễ dàng thích nghi
- Đáp ứng được nhiều request: có khả năng sinh ra hàng chục ngàn tới hàng trăm ngàn ID trong một giây mà không bị trùng

Năm 2010, Twitter lần đầu đưa ra thuật toán tạo ID có thể đáp ứng được các tiêu chí trên, được gọi là Snowflake ID. Link bài viết ở đây: [link](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake)

## Tìm hiểu về Snowflake ID

Thuật toán này quy định giá trị ID được sinh ra là một số nguyên dương 64 bit. Trong đó:

- Bit đầu tiên là số 0, là bit dấu
- 41 bits kế tiếp là chỉ thời gian theo epoch, tính bằng mili giây. Có thể hiểu thời gian theo epoch là khoảng thời gian tính từ thời điểm hiện tại đến một thời điểm nào đó trong quá khứ được chọn làm mốc. Thông thường mốc sẽ là 0 giờ 0 phút 0 giây của một ngày nào đó theo múi giờ UTC. Đối với việc con số chính xác tới giây, 41 bits này thể hiện được thời gian cho khoảng 69 năm kể từ khi bắt đầu epoch
- 10 bits tiếp theo là ID của các server. Các ID này là cố định khi bắt đầu vận hành hệ thống
- 12-bit số đếm từ 0-4095

Theo thuật toán này các trường hợp bị trùng hoặc không hoạt động sẽ là:

- Thời gian tồn tại của thuật toán lâu hơn 69 năm ((2^41-1)/86400000/365)
- Có nhiều hơn 1024 server
- Có nhiều hơn 4096 request tạo ID trong một mili giây, tức là hơn 4.096.000 request tạo ID trong một giây

Số được sinh ra bởi thuật toán này sẽ đảm bảo có thể sắp xếp theo trình tự thời gian

Dựa trên ý tưởng sinh ID như vậy, tùy vào hệ thống mà có cách sắp xếp lại các bit khác nhau. Và dưới đây là một số biến thể của nó.

## Baidu UID

Baidu đã sắp xếp lại 63 bit như sau:

- Bit đầu tiên là bit dấu, mặc định là số 0
- 28 bit tiếp theo là thời gian, tính theo giây. Như vậy thời gian mà nó thể hiện được là khoảng 8.5 năm ((2^28-1)/86400000/365) kể từ lúc bắt đầu epoch, khá ngắn so với quy mô của công ty Baidu
- 22 bit để đánh dấu ID của các server
- 13 bit số đếm

Link tham khảo thuật toán của Baidu ở đây:  [https://github.com/baidu/uid-generator](https://github.com/baidu/uid-generator)

## Sonyflake

Trong khi đó, Sony lại sắp xếp 63 bit như sau:

- Bit đầu tiên là bit dấu, mặc định là số 0
- 39 bit tiếp theo là thời gian, tính theo 10 mili giây (xen-ti giây - centisecond). Với cách biểu diễn này sẽ biểu diễn được 174 năm ((2^39-1)/8640000/365) kể từ lúc bắt đầu epoch, dài hơn rất nhiều so với Snowflake ID
- 8 bits số thứ tự
- 16 bits đánh dấu ID của server

Link tham khảo thuật toán ở đây: [https://github.com/sony/sonyflake](https://github.com/sony/sonyflake)

## Kết luận

Đặc điểm chung của các biến thể của Snowflake ID là sẽ xếp các bit thời gian đứng đầu, như vậy khi tạo ra các số sẽ dễ sắp xếp theo thứ tự thời gian

Tùy vào quy mô hệ thống của mỗi dự án mà sẽ có chiến lược sắp xếp các bit cho phù hợp, hoặc thậm chí là tăng/giảm số bit

## References

[https://www.callicoder.com/distributed-unique-id-sequence-number-generator/](https://www.callicoder.com/distributed-unique-id-sequence-number-generator/) 

[https://blog.devgenius.io/7-famous-approaches-to-generate-distributed-id-with-comparison-table-af89afe4601f](https://blog.devgenius.io/7-famous-approaches-to-generate-distributed-id-with-comparison-table-af89afe4601f)

[https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake)