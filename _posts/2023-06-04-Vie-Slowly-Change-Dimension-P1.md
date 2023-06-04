---
title: 'Slowly Change Dimension trong Data Warehouse - Phần 1/2'
date: 2023-06-04 21:00:00 +0700
categories: []
tags: [Data Warehouse]
pin: false
---
## Giới thiệu

Giả sử ta có một data warehouse được thiết kế theo mô hình Star Schema, tức là lúc này tất cả dữ liệu chỉ nằm trong bảng Dimension hoặc Fact.

Câu hỏi đặt ra là khi dữ liệu trong bảng dimension được cập nhật dữ liệu mới, chúng ta nên xử lý như thế nào. Trong thực tế, các giá trị trong bảng dimensin không hề cố định (trừ bảng dimensin time và dimensin date) mà chúng có khả năng sẽ được cập nhật, nhưng với tần suất không nhiều bằng dữ liệu trong bảng fact. Trong chuyên môn, bảng dimension có dữ liệu được cập nhật theo thời gian như vậy được gọi là Slowly Change Dimension (SCD). Bài viết này sẽ giới thiệu một số cách tổ chức SCD.

Trong nội dung của quyển sách “**The Data Warehouse Toolkit: The Definitive Guide to Dimensional Modeling**” của tác giả Ralph Kimball và Margy Ross thì có các cách tổ chức SCD như sau:

- Loại 1: Ghi đè (Overwrite)
- Loại 2: Thêm dữ liệu mới (Add New Row)
- Loại 3: Thêm thuộc tính mới (Add New Attribute)
- Loại 4: Tạo thêm bảng dimension (Add Mini-Dimension)
- Loại hỗn hợp

## Loại 1: Ghi đè dữ liệu

Đúng như tên gọi, khi cần update dữ liệu bảng dimension, ta sẽ update trực tiếp vào dữ liệu đã có bên trong bảng

Ví dụ:

Ban đầu bảng store dimension chứa dữ liệu về các cửa hàng

| ID | Store Name | Address |
| --- | --- | --- |
| 1 | Store 1 | Address 1 |
| 2 | Store 2 | Address 2 |
| 3 | Store 3 | Address 3 |

Sau đó, vì lý do sáp nhập, đổi chủ nên Store 1 cần được đổi tên thành Store 4

Như vậy bảng dimension mới sẽ là

| ID | Store Name | Address |
| --- | --- | --- |
| 1 | Store 4 | Address 1 |
| 2 | Store 2 | Address 2 |
| 3 | Store 3 | Address 3 |

Cách này làm cho kích thước bảng dimension không thay đổi, nhưng có vấn đề là nó đã bị mất đi lịch sử, một tính chất quan trọng của data warehouse. Giả sử muốn lập một báo cáo doanh thu theo cửa hàng trước và sau khi đổi tên từ Store 1 thành Store 4 thì sẽ không làm được. Mà thực tế thì cũng sẽ sai khi nói rằng Store 4 đã đạt doanh số xxx tỷ đồng trong 1 năm vừa qua trong khi nó chỉ vừa được đổi tên trong thời gian ngắn vừa qua

## Loại 2: Thêm dữ liệu mới

Để khắc phục nhược điểm của loại 1, loại 2 sẽ tổ chức bảng dimension bằng cách thêm một dòng mới vào bảng dimension và đánh dấu rằng dữ liệu nào đang là dữ liệu mới nhất

Như vậy lấy ví dụ ở loại 1 đã nêu, nên tổ chức bảng dimension gốc như sau:

| ID | Store Name | Address | Row Effective Date | Row Expiration Date | Is Current |
| --- | --- | --- | --- | --- | --- |
| 1 | Store 1 | Address 1 | 2020-01-01 | 9999-12-31 | True |
| … | … | … | … | … | … |

Với cách tổ chức này, thì khi cần đổi tên Store 1 thành Store 4 ta sẽ làm như sau

| ID | Store Name | Address | Row Effective Date | Row Expiration Date | Is Current |
| --- | --- | --- | --- | --- | --- |
| 1 | Store 1 | Address 1 | 2020-01-01 | 2023-04-30 | False |
| … | … | … | … | … | … |
| 5 | Store 4 | Address 1 | 2023-05-01 | 9999-12-31 | True |

Như vậy, kể từ ngày 2023-05-01 thì các dữ liệu bảng fact sẽ dùng ID là 5 thay vì là 1 khi tham chiếu tới bảng Store Dimension

Cách tổ chức này khắc phục được nhược điểm của bảng 1 vì nó có lưu được lịch sử, báo cáo doanh thu cũng sẽ chính xác hơn vì nó tách biệt các giao dịch Store 1 và Store 4

Ở đây ta có thể xác định 2 dòng dữ liệu là của cùng 1 cửa hàng thông qua natural key, ở đây có thể là địa chỉ hoặc số điện thoại hoặc mã số thuế. Mình sẽ có một bài viết để nói về các key trong database, data warehouse

Chú ý, tại sao nên có cột ngày Row Expiration Date (ngày đánh dấu dữ liệu hết hiệu lực) trong khi đã có cột ngày Row Effective Date (ngày mà dữ liệu bắt đầu có hiệu lực)? Vì khi tổ chức như vậy thì câu query để tìm ra các store đang hoạt động sẽ có dạng

```sql
SELECT * 
FROM dimension_store
WHERE current_date <= row_expiration_date
```

Sẽ đơn giản hơn nhiều so với việc một câu query đi tìm Store có giá trị Row Effective Date lớn nhất. Hơn nữa lại còn cung cấp trực tiếp thông tin rằng, cửa hàng 1 đã bị đổi tên từ ngày nào.

Tương tự, chức năng của cột Is Current cũng làm cho việc viết query đơn giản hơn, cũng như  đọc vào dễ hiểu hơn, nếu so với việc đọc cột Row Expiration Date

## Loại 3: Thêm thuộc tính mới

Mặc dù loại 2 đã có thể lưu giữ các lịch sử thay đổi, nhưng để tạo lập báo cáo liền mạch giữa các store. Đôi khi với mỗi thay đổi nhỏ của cửa hàng cũng sẽ dẫn đến việc tạo một dòng mới trong bảng dimension, như vậy việc lập báo cáo cho cửa hàng đó sẽ dẫn đến bị gián đoạn vì lúc này store đã có nhiều ID khác nhau mỗi lần thêm dữ liệu mới và update trạng thái dữ liệu cũ

Ở loại 3 này, thay vì thêm vào một dòng mới vào bảng dimension, chúng ta lại thêm một cột mới

Ví dụ: Khi Store 1 muốn đổi tên thành Store 4. Thì bảng dimension sẽ được thiết kế và có giá trị như sau:

| ID | Store Name | Address | Previous Store Name |
| --- | --- | --- | --- |
| 1 | Store 4 | Address 1 | Store 1 |
| … | … | … |  |

Cách này trong thực tế không được sử dụng phổ biến như loại 2. Vì nếu thay vì dữ liệu được update không phải là tên, mà là cột khác, thì ta sẽ phải thêm một cột mới thay vì là cột Previous Store Name. Hơn nữa, nó chỉ ghi nhận được 1 lần thay đổi, nếu có nhiều thay đổi thì sẽ phải tạo nhiều cột. Bảng dimension lúc này sẽ bị phình ra theo chiều ngang mà có rất nhiều ô bị NULL (vì chỉ có số ít cửa hàng đổi tên, chứ không phải toàn bộ cửa hàng). Trong khi để khắc phục nhược điểm của loại 2 thì ta chỉ cần thêm một cột để thể hiện giá trị duy nhất với mỗi cửa hàng (ví dụ mã số thuế)

Chỉ nên sử dụng loại 3 khi thuộc tính được cập nhật với số lần hữu hạn

## Loại 4: Tạo thêm bảng dimension

Với các bảng dimension có nhiều dòng dữ liệu, ví dụ như Product Dimension để lưu dữ liệu của các sản phẩm được bán của một cửa hàng. Bảng dimension này hoàn toàn có thể đạt tới kích thước lớn với số lượng hàng triệu dòng. Việc áp dụng loại 2 tức là sẽ tạo ra một dòng dữ liệu mới và update lại dữ liệu cũ trong một bảng hàng triệu dòng như vậy không phải là một cách hay

Loại 4 đưa ra cách giải quyết rằng, các dữ liệu trong bảng dimension vẫn sẽ được dùng để lưu các giá trị mới nhất, nhưng các giá trị lịch sử sẽ được lưu trong một bảng dimension khác

Như vậy, với việc Store 1 đổi tên thành Store 4, theo như loại 4 sẽ có 2 bảng được update như sau:

Bảng Store Dimension

| ID | Store Name | Address |
| --- | --- | --- |
| 1 | Store 4 | Address 1 |
| 2 | Store 2 | Address 2 |
| 3 | Store 3 | Address 3 |

Bảng Store History Dimension

| ID | Store ID | Store Name | Address | Created Date |
| --- | --- | --- | --- | --- |
| 1 | 1 | Store 1 | Address 1 | 2020-01-01 |
| 2 | 1 | Store 4 | Address 1 | 2023-05-01 |

Bài viết đã dài nên mình xin tạm dừng ở đây. Phần tiếp theo sẽ giới thiệu cách kết hợp các loại với nhau nhé