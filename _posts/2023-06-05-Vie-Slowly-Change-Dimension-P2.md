---
title: 'Slowly Change Dimension trong Data Warehouse - Phần 2/2'
date: 2023-06-05 16:30:00 +0700
categories: [Theory, Data Warehouse]
tags: [Data Warehouse]
pin: false
---

## Giới thiệu

Ở bài viết trước ([Slowly Change Dimension trong Data Warehouse - Phần 1/2](https://vinhtran1.github.io/posts/Vie-Slowly-Change-Dimension-P1/)) đã nói về các cách tổ chức SCD khác nhau. Trong thực tế sẽ có các trường hợp không sử dụng đơn lẻ từng loại mà là kết hợp nhiều loại với nhau. Ở bài viết này mình sẽ giới thiệu một số cách kết hợp của các cách đã giới thiệu ở phần 1

## Loại 5: 1 + 4

Loại 5 đưa ra cách giải quyết rằng, các dữ liệu trong bảng Dimension vẫn sẽ được dùng để lưu các giá trị mới nhất, nhưng các giá trị lịch sử sẽ được lưu trong một bảng Dimension khác

Như vậy, với việc **Store Name 1** đổi tên thành **Store Name 4**, theo như loại 4 sẽ có 2 bảng được update như sau:

Bảng `Mini Store Dimension` sẽ được thêm vào một dòng mới

| ID | Store Name | Attribute 1 | … | Attrbute N |
| --- | --- | --- | --- | --- |
| 1 | Store Name 1 | … | … | … |
| 2 | Store Name 4 | … | … | … |

Sau khi có ID của các dữ liệu vừa được thêm vào, cập nhật lại ID ở cột **Current Mini Store Dimension ID** của bảng `Store Dimension`

| ID | Store Code | Address | Current Mini Store Dimension ID |
| --- | --- | --- | --- |
| 1 | StoreCode1 | Address 1 | 2 |
| … | … | … | … |

Cứ như vậy, mỗi khi dữ liệu cửa hàng được thay đổi, thì bảng `Mini Store Dimension` sẽ thêm dòng mới (loại 4), sau đó cập nhật lại ID vào cột **Current Mini Store Dimension ID** của bảng `Store Dimension` (loại 1)

![SCD type 5.png](https://images2.imgbox.com/b9/8e/CKR19inN_o.png)

## Loại 6: 1+2+3

Với loại 6 này, mỗi khi có một thay đổi dữ liệu bảng Dimension thì sẽ có 2 thao tác được thực hiện

- Tạo một dòng dữ liệu mới trong bảng Dimension và đánh dấu dòng đó là dữ liệu mới nhất
- Tạo thêm thuộc tính để chứa giá trị hiện tại và câp nhật các giá trị cũ trong bảng Dimension thành giá trị hiện tại

Để đổi tên **Store Name 1** thành **Store Name 4** thì cấu trúc bảng Dimension sẽ như sau

Bảng `Store Dimension` ban đầu:

| ID | Store Name | Store Code | Address | … |
| --- | --- | --- | --- | --- |
| 1 | Store Name 1 | StoreCode1 | Address 1 | … |
| 2 | Store Name 2 | StoreCode2 | Address 2 | ... |
| 3 | Store Name 3 | StoreCode3 | Address 3 | … |

Bảng `Store Dimension` sau khi đổi tên **Store Name 1** thành **Store Name 4** có dữ liệu được cập nhật

| ID | Store Name | Current Store Name | Store Code | Address | Row Effective Date | Row Expiration Date | Is Current | … |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Store Name 1 | Store Name 4 | StoreCode1 | Address 1 | 2020-01-01 | 2023-04-30 | False | … |
| 2 | … | … | … | … | … | … | … | ... |
| 3 | Store Name 4 | Store Name 4 | StoreCode1 | Address 1 | 2023-05-01 | 9999-12-31 | True | … |

Sau một thời gian, nếu **Store Name 4** muốn đổi tên thành **Store Name 10** chẳng hạn, thì bảng `Store Dimension` sẽ là:

| ID | Store Name | Current Store Name | Store Code | Address | Row Effective Date | Row Expiration Date | Is Current | … |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Store Name 1 | Store Name 10 | StoreCode1 | Address 1 | 2020-01-01 | 2023-04-30 | False | … |
| 2 | … | … | … | … | … | … | … | ... |
| 3 | Store Name 4 | Store Name 10 | StoreCode2 | Address 1 | 2023-05-01 | 2024-05-31 | False | … |
| 4 | Store Name 10 | Store Name 10 | StoreCode3 | Address 1 | 2024-06-01 | 9999-12-31 | True | … |

Như vậy với cách kết hợp này, ta đã 

- Insert một dòng mới vào bảng Dimension để xác định dữ liệu mới nhất (SCD loại 2)
- Thêm một cột mới để xác định giá trị hiện tại (SCD loại 3)
- Update lại toàn bộ giá trị cũ bằng giá trị mới nếu có thay đổi thêm lần nữa (SCD loại 1)

Kỹ thuật này giúp cho việc ghi nhận dữ liệu hiện tại và dữ liệu lịch sử trở nên rất rõ ràng nhưng cách thiết kế lại khá phức tạp

## Loại 7: 1 + 2

SCD loại 7 có ý tưởng thiết kế rằng khi một dữ liệu được thay đổi, dữ liệu lịch sử và hiện tại sẽ lưu lại ở 2 bảng Dimension riêng biệt:

- Thêm dữ liệu mới vào bảng `History Store Dimension`, đánh dấu rằng dòng dữ liệu là mới nhất. Bảng này sẽ khởi tạo một ID mới là khóa chính, **cộng thêm một** **Durable Supernatural Key**
- Cập nhật dữ liệu mới nhất vào bảng `Current Store Dimension`. Bảng này sẽ **lưu** **Durable Supernatural Key làm khóa chính** và chỉ lưu giá trị hiện tại. Hoặc cách khác có thể thay thế bằng việc tạo một View (xem khái niệm View, Materialized View trong database) chứa dữ liệu có đánh dấu **Is Current = True** trong bảng `History Store Dimension`
- Bảng fact sẽ tham chiếu đến cả 2 bảng `History Store Dimension` và `Current Store Dimension`, và lưu trữ cả khóa chính của 2 bảng

Kết quả SCD loại 7 sẽ như hình dưới đây

![SCD type 7.png](https://images2.imgbox.com/88/c7/KUet7RT0_o.png)

Việc tách ra thành 2 bảng sẽ làm cho bảng `Current Store Dimension` sẽ có ít dòng hơn nhiều so với bảng `History Store Dimension`. Như vậy cách này sẽ thích hợp với nhu cầu thường xuyên báo cáo với các giá trị hiện tại hơn là các giá trị lịch sử

## Tổng kết

Dưới đây là bảng tóm tắt về SCD các loại

| Loại SCD | Kỹ thuật áp dụng |
| --- | --- |
| Loại 1 | Ghi dè lên giá trị cũ mỗi khi có giá trị mới cần được cập nhật |
| Loại 2 | Thêm một dòng dữ liệu mới vào và đánh dấu dữ liệu này là mới nhất |
| Loại 3 | Tạo thêm một cột mới bên trong bảng Dimension để lưu lại giá trị cũ trước khi cập nhật |
| Loại 4 | Tạo một bảng phụ Mini-Dimension để lưu các thuộc tính thường xuyên thay đổi |
| Loại 5 | Như Loại 4 nhưng có thêm bước cập nhật giá trị mới nhất vào bảng Dimension chính (loại 1) |
| Loại 6 | Thêm một dòng mới để lưu giá trị mới nhất (loại 2), thêm cột mới để ghi nhận giá trị hiện tại (loại 3), cập nhật cột mới bằng giá trị mới trên toàn bộ dữ liệu cũ có liên quan (loại 1) |
| Loại 7 | Tạo một bảng Current Dimension để lưu lại giá trị mới nhất, song song với bảng History Dmension để lưu các giá trị mới nhất + giá trị lịch sử |

Theo kinh nghiệm thì khi mới bắt đầu thiết kế SCD thì sẽ bắt đầu bằng SCD loại 2 vì nó không quá khó setup, và lưu được lịch sử. Sau đó sẽ tùy từng nhu cầu mà chọn cách thiết kế SCD khác. Việc chọn lựa bất kỳ một loại SCD sẽ có sự đánh đổi giữa các yếu tố sau đây:

- Dễ thiết kế
- Câu query đơn giản
- Performance tốt
- Lưu được các dữ liệu lịch sử

Được cái này thì sẽ mất cái kia.

Hy vọng bài viết có ích với mọi người.