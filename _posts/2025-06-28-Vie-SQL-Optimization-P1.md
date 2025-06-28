---
title: 'SQL Optimization - Bài 1 - Tối ưu truy vấn ngắn - Index'
date: 2025-06-28 16:30:00 +0700
categories: [Theory, Data Warehouse]
tags: [Data Warehouse]
pin: false
---

![](https://images2.imgbox.com/4d/8c/4kr3WD6S_o.png)

# 1. Mở đầu

Đây là bài thứ 2 trong chuỗi bài viết về SQL Optimization. Trong bài này sẽ nói về kỹ thuật để tối ưu truy vấn ngắn

Nhắc lại, truy vấn ngắn là các truy vấn chỉ sử dụng một lượng nhỏ dữ liệu từ các bảng thành phần để tính toán ra kết quả, thông thường là dưới 10%

# 2. Index

Nguyên tắc chủ yếu khi gặp truy vấn ngắn là phải nhanh chóng chọn được những dữ liệu cần thiết mà không phải quét toàn bộ bảng (Full Scan). Với yêu cầu này thì không có gì phù hợp hơn là dùng Index.

Chúng ta có một chỉ số gọi là **Index Selectivity** để đo lường mức độ hiệu quả của một Index trong việc nhanh chóng tìm ra những dữ liệu cần thiết. Những cột có tính “duy nhất” càng cao thì sử dụng index càng có lợi. Một cột 10 triệu dòng nhưng chỉ có 2 giá trị là 0 và 1 thì rõ ràng không nên sử dụng index. Chính vì vậy mà PostgreSQL sẽ tự động tạo index khi bạn tạo một constraint Primary key hoặc unique key vì rõ ràng những cột này có tính duy nhất cực kỳ cao.

Khi mệnh đề WHERE hoặc JOIN… ON… trong câu lệnh SQL, Optimizer sẽ bắt đầu từ điều kiện có index với tính duy nhất rất cao

Có một lưu ý là có một số Cơ sở dữ liệu không hỗ trợ index scan cho những dòng không chứa dữ liệu (NULL value) (tính tới thời điểm viết bài thì Oracle và MySQL InnoDB không hỗ trợ, trong khi PostgreSQL và SQL Server thì có). Vì vậy nếu cần thiết phải truy vấn đến những dữ liệu đó thường xuyên thì có 2 cách để xử lý:

1. Tạo một index cho cột đó với điều kiện is null
2. Thay những ô bị null thành một giá trị mặc định (ví dụ -1 đối với số, ‘1900-01-01’ đối với ngày tháng, ‘null’ đối với chuỗi). Recommend cách này hơn vì không cần phải tốn công nhớ và kiểm tra xem cơ sở dữ liệu có hỗ trợ index trên nuill không, cũng không cần phải tốn thêm thời gian công sức để tạo index thứ 2 trên cùng một cột. Hơn nữa để tránh trường hợp khi join các bảng với nhau mà gặp null thì sẽ phân biệt được khi nào null do bản chất dữ liệu, khi nào null do phép join không tìm thấy record tương ứng

Mọi người có thể sử dụng dữ liệu được gắn trong link này để import vào PostgreSQL để thực hành. Hãy yên tâm vì tất cả đều là dữ liệu random, không có dòng nào đại diện cho dữ liệu thật

# 3. Thực hành: Index không hoạt động khi có biến đổi trên cột

Ta sẽ tạo một Index trên cột `date_of_birth` của bảng clients

```sql
CREATE INDEX clients_date_of_birth_idx 
	ON test.clients USING btree (date_of_birth)
```

Ví dụ muốn tìm những client có độ tuổi lớn hơn 100 tuổi. Nếu mọi người có ý định viết SQL theo logic: nếu năm hiện tại trừ đi năm sinh lớn hơn 100 tức là client đó đã hơn 100 tuổi. Tức là

```sql
  
explain
select *
from test.clients c 
where extract(year from current_date) -  extract(year from date_of_birth) > 100
```

Thì kết quả explain sẽ là

```
QUERY PLAN                                                                                       |
-------------------------------------------------------------------------------------------------+
Seq Scan on clients c  (cost=0.00..4370.00 rows=33333 width=130)                                 |
  Filter: ((EXTRACT(year FROM CURRENT_DATE) - EXTRACT(year FROM date_of_birth)) > '100'::numeric)|
```

Optimizer sẽ lựa chọn Seq scan vì lúc này nó không tìm kiếm trên cột `date_of_birth` mà chỉ là cột biến đổi của nó `extract(year from date_of_birth)`. Và rõ ràng là cột mới biến đổi này chưa được tạo index

Thay vào đó, nếu ta chính sửa câu SQL một tí

```sql

explain  
select *
from test.clients c 
where date_of_birth < '1926-01-01 00:00:00'::timestamp
```

Thì kết quả sẽ là:

```
QUERY PLAN                                                                                 |
-------------------------------------------------------------------------------------------+
Index Scan using clients_date_of_birth_idx on clients c  (cost=0.29..8.31 rows=1 width=130)|
  Index Cond: (date_of_birth < '1926-01-01 00:00:00'::timestamp without time zone)         |
```

Optimizer sẽ ưu tiên quét qua Index trước

Thực ra do điều kiện này rất hiếm, tức là hiếm có client nào trên 100 tuổi trong thực tế nên selectivity của nó rất cao nên optimizer mới lựa chọn Index Scan. Nếu ta đổi lại điều kiện lọc là lớn hơn 70 tuổi. Tức là sinh trước ngày 01/01/1956 thì lúc này kết quả explain sẽ là:

```
QUERY PLAN                                                                                  |
--------------------------------------------------------------------------------------------+
Bitmap Heap Scan on clients c  (cost=424.44..2912.41 rows=29438 width=130)                  |
  Recheck Cond: (date_of_birth < '1956-01-01 00:00:00'::timestamp without time zone)        |
  ->  Bitmap Index Scan on clients_date_of_birth_idx  (cost=0.00..417.08 rows=29438 width=0)|
        Index Cond: (date_of_birth < '1956-01-01 00:00:00'::timestamp without time zone)    |
```

Và nếu thay đổi điều kiện rằng tìm những client trên 30 tuổi thì kết quả sẽ là

```
QUERY PLAN                                                                    |
------------------------------------------------------------------------------+
Seq Scan on clients c  (cost=0.00..3370.00 rows=84126 width=130)              |
  Filter: (date_of_birth < '1996-01-01 00:00:00'::timestamp without time zone)|
```

Lúc này, optimizer cho rằng quét toàn bộ bảng mang lại tốc độ tốt hơn vì tính unique của dữ liệu cần chọn ra lúc này không còn cao nữa, rõ ràng có rất nhiều client trên 30 tuổi. Thực tế trong data mẫu của mình thì có đến hơn 84k dòng dữ liệu thỏa điều kiện

# 4. Vậy có những cách viết SQL như thế nào để tối đa sức mạnh của index

## 4.1 Case sensitive đối với chuỗi

Đối với trường hợp cần tìm kiếm trên chuỗi, ta sẽ thường chuẩn hóa về một kiểu viết thường rồi sau đó sẽ tìm kiếm. Cụ thể là

```sql
WHERE lower(col) = 'value';
```

Cách giải quyết sẽ là tạo index trên chuỗi chuẩn hóa. Cụ thể trong trường hợp trên sẽ là:

```sql
CREATE INDEX idx_lower_col ON account (lower(col));
```

Như vậy câu SQL trên sẽ sử dụng index khi thực thi. Thật ra kỹ thuật này chỉ khi tìm hiểu mình mới biết, không nghĩ nó lại đơn giản như vậy

## 4.2 Thay đổi cách viết: datetime, timestamp

Mình rất thường gặp câu SQL có cách viết thế này khi cần lấy những dữ liệu có sự thay đổi trong vòng 1 ngày

```sql
WHERE created_time::date >= current_date - interval '1' day
```

Cách viết này có vấn đề ở chỗ dùng nhiều hàm biến đổi (cast, current_date, interval). Như đã phân tích, cách viết này không sử dụng được index. Nên viết lại như sau:

```sql
with cte_yesterday as (
	select (current_date - interval '1' day)::timestamp as yesterday
)
select ....
WHERE created_time >= (select yesterday from cte_yesterday)
```

Cách viết mới có thể dài hơn, nhưng hoàn toàn không dùng hàm biến đổi trên cột ở điều kiện WHERE. Điều này chắc chắn sẽ giúp câu truy vấn nhanh hơn nhiều

## 4.3 Coalesce cũng là hàm biến đổi

Coalesce cũng là hàm làm biến đổi cột, nên hạn chế dùng nếu cột đó có index

Cách biến đổi là dùng OR … IS NULL

```sql
select * 
from tableA
where coalesce(date_column1, date_column2) between '2020-08-17' AND '2020-08-18'
```

Ta có thể thay đổi cách viết lại như sau:

```sql
select * 
from tableA
where (date_column1 between '2020-08-17' AND '2020-08-18')
OR (date_column1 IS NULL AND date_column2 between '2020-08-17' AND '2020-08-18')
```

lúc này chỉ date_column1, date_column2 sẽ sử dụng index scan

# 5. Lesson Learned

Mình tạm dừng bài viết tại đây, sẽ tiếp tục chủ đề này vào bài tiếp theo vì nội dung còn khá dài, nếu chỉ gói gọn trong 1 bài viết thì không đủ

Leasson learned:

1. Tránh tạo index trên những cột có selectivity thấp
2. Unique key và primary key được mặc định tạo index
3. Chú ý đến những cột nullable vì có thể index không hoạt động khi tìm kiếm IS NULL
4. Hạn chế (thậm chí là không) dùng hàm nhằm biến đổi đến những cột ở mệnh đề WHERE, JOIN… ON… đặc biệt là khi cột đó đã được tạo index
