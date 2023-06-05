---
title: 'Slowly Change Dimension trong Data Warehouse - Phần 1/2'
date: 2023-06-05 16:25:00 +0700
categories: [Theory, Data Warehouse]
tags: [Data Warehouse]
pin: false
---

## Giới thiệu

Giả sử ta có một Data Warehouse được thiết kế theo mô hình Star Schema, tức là lúc này tất cả dữ liệu chỉ nằm trong bảng Dimension hoặc Fact.

Câu hỏi đặt ra là khi dữ liệu trong bảng Dimension được cập nhật dữ liệu mới, chúng ta nên xử lý như thế nào. Trong thực tế, các giá trị trong bảng dimensin không hề cố định (trừ bảng `Time Dimension` và `Date Dimension`) mà chúng có khả năng sẽ được cập nhật, nhưng với tần suất không thường xuyên và không dự đoán trước được.

Trong chuyên môn, bảng Dimension có dữ liệu được cập nhật theo thời gian như vậy được gọi là **Slowly Change Dimension** (SCD). Bài viết này sẽ giới thiệu một số cách tổ chức SCD.

Trong nội dung của quyển sách “**The Data Warehouse Toolkit: The Definitive Guide to Dimensional Modeling**” của tác giả Ralph Kimball và Margy Ross thì có các cách tổ chức SCD như sau:

- Loại 1: Ghi đè (Overwrite)
- Loại 2: Thêm dữ liệu mới (Add New Row)
- Loại 3: Thêm thuộc tính mới (Add New Attribute)
- Loại 4: Tạo thêm bảng Mini Dimension (Add Mini-Dimension)
- Loại hỗn hợp (Hybrid Techniques)

Trong phần 1 sẽ giới thiệu về 4 loại SCD cơ bản

## Loại 1: Ghi đè dữ liệu

Đúng như tên gọi, khi cần update dữ liệu bảng Dimension, ta sẽ update trực tiếp vào dữ liệu đã có bên trong bảng

**Ví dụ:**

Ban đầu bảng `Store Dimension` chứa dữ liệu về các cửa hàng gồm có khóa chính, tên cửa hàng, địa chỉ,…

| ID | Store Name | Store Code | Address | … |
| --- | --- | --- | --- | --- |
| 1 | Store Name 1 | StoreCode1 | Address 1 | … |
| 2 | Store Name 2 | StoreCode2 | Address 2 | ... |
| 3 | Store Name 3 | StoreCode3 | Address 3 | … |

Sau đó, vì lý do sáp nhập, đổi chủ, hoặc đơn giản là đổi tên cho hợp phong thủy hơn nên **Store Name 1** cần được đổi tên thành **Store Name 4**

Như vậy bảng Dimension mới sẽ là

| ID | Store Name | Store Code | Address |
| --- | --- | --- | --- |
| 1 | Store Name 4 | StoreCode1 | Address 1 |
| 2 | Store Name 2 | StoreCode2 | Address 2 |
| 3 | Store Name 3 | StoreCode3 | Address 3 |

Cách này làm cho kích thước bảng Dimension không thay đổi, nhưng có vấn đề là nó đã bị mất đi lịch sử, một tính chất quan trọng của Data Warehouse. Giả sử muốn lập một báo cáo doanh thu theo cửa hàng trước và sau khi đổi tên từ **Store Name 1** thành **Store Name 4** thì sẽ không làm được. Mà thực tế thì cũng sẽ sai khi nói rằng **Store Name 4** đã đạt doanh số **XYZ** tỷ đồng trong 1 năm vừa qua trong khi nó chỉ vừa được đổi tên trong thời gian ngắn vừa qua

## Loại 2: Thêm dữ liệu mới

Để khắc phục nhược điểm của loại 1, loại 2 sẽ tổ chức bảng Dimension bằng cách thêm một dòng mới vào bảng Dimension và đánh dấu rằng dữ liệu nào đang là dữ liệu mới nhất

Như vậy lấy ví dụ ở loại 1 đã nêu, nên tổ chức bảng Dimension gốc như sau:

| ID | Store Name | Store Code | Address | Row Effective Date | Row Expiration Date | Is Current |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Store Name 1 | StoreCode1 | Address 1 | 2020-01-01 | 9999-12-31 | True |
| … | … |  | … | … | … | … |

Với cách tổ chức này, thì khi cần đổi tên **Store Name** **1** thành **Store Name** **4** ta sẽ làm như sau

| ID | Store Name | Store Code | Address | Row Effective Date | Row Expiration Date | Is Current |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Store Name 1 | StoreCode1 | Address 1 | 2020-01-01 | 2023-04-30 | False |
| … | … | … | … | … | … | … |
| 5 | Store Name 4 | StoreCode1 | Address 1 | 2023-05-01 | 9999-12-31 | True |

Như vậy, kể từ ngày **2023-05-01** thì các dữ liệu bảng fact sẽ dùng ID là 5 thay vì là 1 khi tham chiếu tới bảng `Store Dimension`

Cách tổ chức này khắc phục được nhược điểm của bảng 1 vì nó có lưu được lịch sử, báo cáo doanh thu cũng sẽ chính xác hơn vì nó tách biệt các giao dịch **Store Name 1** và **Store Name 4**

**Lưu ý 1:**

Tại sao nên có cột ngày **Row Expiration Date** (ngày đánh dấu dữ liệu hết hiệu lực) trong khi đã có cột ngày **Row Effective Date** (ngày mà dữ liệu bắt đầu có hiệu lực)? Vì khi tổ chức như vậy thì câu query để tìm ra các store đang hoạt động sẽ có dạng:

```sql
SELECT * 
FROM "Store Dimension"
WHERE current_date between "Row Effective Date" and "Row Expiration Date";
```

câu query này sẽ đơn giản hơn nhiều so với việc một câu query đi tìm cửa hàng có giá trị **Row Effective Date** lớn nhất. Hơn nữa lại còn cung cấp trực tiếp thông tin rằng, cửa hàng có tên **Store Name 1** đã bị đổi tên từ ngày nào.

Tương tự, chức năng của cột **Is Current** cũng làm cho việc viết query đơn giản hơn, cũng như  đọc vào dễ hiểu hơn, nếu so với việc đọc cột **Row Expiration Date**

**Lưu ý 2**:

Khi tất cả dữ liệu của một cửa hàng có thể bị thay đổi: tên cửa hàng, địa chỉ, chủ sỡ hữu,… tuy nhiên vẫn có một dữ liệu không thay đổi, nó vẫn gắn liền với cửa hàng đó. Cột đó là cột **Store Code**. Ta gọi cột này là một **Durable Supernatural Key** (hoặc có tên khác là **Durable Key**. Tạm dịch: Khóa bền). Mình sẽ có một bài viết về các loại khóa trong database sau.

Cột này có ý nghĩa quan trọng đối với SCD loại 2 và các SCD loại hỗn hợp có áp dụng SCD loại 2. Nhờ cột này mà việc lập báo cáo không bị gián đoạn bởi bất cứ việc thay đổi dữ liệu nào, thậm chí là thay đổi toàn bộ dữ liệu của cửa hàng đó. Nếu không có cột này. **Store Name 1** và **Store Name 4** dường như được xem là 2 cửa hàng riêng biệt và sẽ gây khó khăn khi viết truy vấn lập báo cáo liền mạch từ lúc **Store Name 1** đến lúc đổi tên thành  **Store Name 4**

## Loại 3: Thêm thuộc tính mới

Đối với SCD loại 2, mặc dù đã lưu được lịch sử thay đổi giá trị, nhưng sẽ khó để truy ra được dữ liệu trước khi được thay đổi của một dòng dữ liệu nằm ở dòng nào (Dĩ nhiên là bỏ công thêm một chút để viết truy vấn thì vẫn được).

Ở SCD loại 3 này sẽ giải quyết cái khó đó của SCD loại 2, thay vì thêm vào một dòng mới vào bảng Dimension, chúng ta lại thêm một cột mới

**Ví dụ**: Khi **Store Name 1** muốn đổi tên thành **Store Name 4,** thì bảng Dimension sẽ được thêm vào cột **Previous Store Name**, kết quả như sau:

| ID | Store Name | Store Code | Address | Previous Store Name |
| --- | --- | --- | --- | --- |
| 1 | Store Name 4 | StoreCode1 | Address 1 | Store Name 1 |
| … | … |  | … |  |

Tuy nhiên, cách này trong thực tế không được sử dụng phổ biến như loại 2 vì nếu thay vì dữ liệu được update không phải là tên, mà là nhiều cột khác, thì ta sẽ phải thêm tương ứng nhiều cột mới **Previous <<Attribute Name>>**. Chưa kể, mỗi cột như thế này chỉ ghi nhận được 1 lần thay đổi, nếu có nhiều thay đổi thì sẽ phải tạo nhiều cột hơn nữa. Kết quả là bảng Dimension lúc này sẽ bị phình ra theo chiều ngang mà có rất nhiều ô bị **Null** (vì chỉ có số ít cửa hàng đổi tên, chứ không phải tất cả cửa hàng).

Vì vậy, chỉ nên sử dụng loại 3 khi thuộc tính được cập nhật với số lần hữu hạn

## Loại 4: Tạo thêm bảng Mini Dimension

Với các bảng Dimension có nhiều dòng dữ liệu, ví dụ như `Product Dimension` để lưu dữ liệu của các sản phẩm được bán của một cửa hàng. Bảng Dimension này hoàn toàn có thể đạt tới kích thước lớn với số lượng hàng triệu dòng. Việc áp dụng SCD loại 2 tức là sẽ tạo ra một dòng dữ liệu mới và update lại dữ liệu cũ trong một bảng hàng triệu dòng như vậy không phải là một cách hay

Loại 4 đưa ra cách giải quyết rằng, những thuộc tính nào có khả năng thay đổi sẽ được tách riêng ra thành một bảng Dimension mới, trong khi các thuộc tính nào (gần như) không thay đổi thì vẫn được lưu trữ trên bảng Dimension gốc 

Như vậy, với việc **Store Name 1** đổi tên thành **Store Name 4**, theo như loại 4 sẽ có 2 bảng được update như sau:

**Bảng Store Dimension**

| ID | Store Code | Address | … |
| --- | --- | --- | --- |
| 1 | StoreCode1 | Address 1 | … |
| 2 | StoreCode2 | Address 2 | … |
| 3 | StoreCode3 | Address 3 | … |

**Bảng Mini Store Dimension**

| ID | Store ID | Store Name |
| --- | --- | --- |
| 1 | 1 | Store Name 1 |
| 2 | 1 | Store Name 4 |

Mối quan hệ giữa các bảng hiện tại sẽ như hình

![SCD type 4.png](https://images2.imgbox.com/af/ae/bTfPnEuE_o.png)

Như vậy, với việc giả định rằng tên cửa hàng là thuộc tính sẽ dễ có cập nhật nhất trong số các thuộc tính của `Store Dimension`, ta sẽ tách cột **Store Name** để lập thành một bảng Dimension mới.

Như vậy là đã xong 4 loại SCD cơ bản. Phần tiếp theo sẽ giới thiệu cách kết hợp các loại SCD với nhau nhé. **Link**: [Slowly Change Dimension trong Data Warehouse - Phần 2/2](https://vinhtran1.github.io/posts/Vie-Slowly-Change-Dimension-P2/)