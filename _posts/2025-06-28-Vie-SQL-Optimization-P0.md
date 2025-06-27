---
title: 'SQL Optimization - Bài 0 - Những khái niệm đầu tiên'
date: 2025-06-28 09:00:00 +0700
categories: [Theory, SQL Optimization]
tags: [SQL]
pin: false
---


![](https://images2.imgbox.com/89/18/tAiO2vsT_o.png)

# I. Mở đầu

Mình xin mở bài ngắn gọn thôi, để viết SQL lấy được đúng data cần thiết là một việc, nhưng biết cách viết để câu SQL có thể chạy nhanh nhất lại là một vấn đề khác, cần bỏ nhiều công sức để nghiên cứu hơn. Trong chuỗi bài viết này mình sẽ tổng hợp lại những kiến thức liên quan đến việc tối ưu câu SQL mà mình đã góp nhặt được

Các ví dụ trong chuỗi bài này sẽ được thực hiện trên cơ sở dữ liệu PostgreSQL, nhưng hoàn toàn có thể dùng cho các hệ quản trị cơ sở dữ liệu khác vì chúng tương tự nhau rất nhiều

# II. Khi chạy một câu lệnh SELECT, CSDL sẽ làm gì

Khi bạn thực thi một câu SQL, PostgreSQL thực hiện nhiều bước có thể tóm gọn thành 3 bước chính như sau:

1. **Biên dịch (Compile):** Chuyển đổi câu lệnh SQL thành một kế hoạch logic (logical plan) gồm các phép toán logic cấp cao.
2. **Tối ưu hóa (Optimize):** Tối ưu hóa kế hoạch logic và chuyển đổi nó thành một kế hoạch thực thi (execution plan) vật lý.
3. **Thực thi (Execute):** Chạy kế hoạch thực thi và trả về kết quả.

## 2.1 Biên dịch (compile)

Có thể bạn đã biết (hoặc sắp biết), SQL là một ngôn ngữ khai báo (Declarative Language). Tức là, những gì bạn viết thể hiện những kết quả bạn muốn thấy mà không cần quan tâm bên dưới thực hiện ra sao. Khác với ngôn ngữ thủ tục (Procedure language), bạn sẽ định nghĩa thuật toán cho máy tính, và máy tính bắt buộc thực hiện theo những gì bạn viết. Vì vậy, mỗi cơ sở dữ liệu đều sẽ có một trình biên dịch câu SQL của chúng ta thành những chỉ dẫn cụ thể cho máy tính hiểu

Việc hiểu ra điều này khá quan trọng, vì trong quá trình làm việc, mình nhận ra có nhiều bạn dùng tư duy của lập trình thủ tục để áp dụng vào viết SQL. Ví dụ, theo bạn câu SQL nào dưới đây sẽ chạy nhanh hơn

Câu 1:

```sql
WITH cte_tableA as (
	select col1A, col2A, col3A, col4A
	from tableA
	where col5A = 'valueA'
)
, cte_tableB as (
	select col1B, col2B, col3B, col4B
	from tableB
	where col5B = 'valueB'
)
select a.*, b.*
from cte_tableA a
join cte_tableB b on a.col1A = b.col1B
```

Câu 2:

```sql
Select a.col1A, a.col2A, a.col3A, a.col4A
			, b.col1B, col2B, col3B, col4B
from tableA a
join tableB b on a.col1A = b.col1B
where a.col5A = 'valueA'
and b.col5B = 'valueB'
```

Câu 1 rõ ràng là muốn ‘Lọc bớt data trước’ để ‘Join sau’. Trong lập trình thủ tục, đây là một tư duy hoàn toàn thực tế giúp code chạy nhanh hơn. Nhưng đối với SQL thì chưa chắc, chúng ta sẽ tìm hiểu chi tiết vào những bài sau.

## 2.2 Tối ưu hóa (Optimize)

Sau khi máy tính đã dịch câu lệnh của chúng ta, nó sẽ kết hợp với những dữ liệu khác có trong Database, ví dụ: 

- Câu SQL cần phải Join bao nhiêu bảng
- Kích thước các bảng như thế nào
- Các bảng được tạo index hay không, trên những cột nào
- …

Kết hợp tất cả dữ kiện trên, CSDL sẽ “vẽ” ra được một thứ tự thực hiện các bước cụ thể, thậm chí là chi phí ước tính để thực hiện từng bước. Thứ tự này gọi là Execution Plan. Trong phần lớn thời gian, câu chuyện tối ưu câu SQL chính là viết SQL và tổ chức CSDL làm sao tối ưu Execution Plan này

Bộ công cụ của CSDL để tối ưu gọi là Query Planner hoặc Query Optimizer

## 2.3 Thực thi (Execute)

Bước này chỉ đơn giản là thực thi đúng trình tự theo plan đã đề ra ở bước trên và trả kết quả về cho người dùng

# III. Truy vấn ngắn và truy vấn dài

Chữ “ngắn” và “dài” không mô tả về độ dài của một câu truy vấn hay độ lớn của dữ liệu trả về.

Truy vấn ngắn (short queries) là những câu truy vấn thường sử dụng một lượng nhỏ dữ liệu so với toàn bộ bảng để tham gia tính toán. Thường thì con số này là dưới 10% lượng dữ liệu từ bảng thành phần. Ngược lại, những truy vấn cần một lượng dữ liệu đáng kể so với bảng thành phần để xử lý dữ liệu thì gọi là truy vấn dài. 

Tại sao lại cần phân biệt như vậy? Vì query planner sẽ lựa chọn cách thực thi khác nhau dựa vào chi phí ước tính ban đầu của nó, trong đó có cả ước tính số dòng dữ liệu cần dùng.

Nguyên tắc chủ yếu khi gặp truy vấn ngắn đó là tránh quét toàn bộ bảng, làm nhiều cách để lấy ra được lượng dữ liệu nhỏ cần thiết trong thời gian nhanh nhất. Còn nguyên tắc chủ yếu khi gặp truy vấn dài là chấp nhận quét toàn bộ bảng, nhưng tránh quét đi quét lại nhiều lần, và cố gắng giảm dữ liệu trong quá trình xử lý trung gian càng sớm càng tốt

# IV. Lesson learned

Như vậy trong bài này ta đã học được các kiến thức đầu tiên của kỹ thuật SQL Optimization

1. SQL là một ngôn ngữ khai báo (Declarative Language). Bạn viết code một kiểu nhưng máy tính có thể thực hiện một kiểu khác
2. Việc tối ưu SQL thực tế chính là một loạt kỹ thuật để tối ưu execution plan
3. Truy vấn ngắn và Truy vấn dài có cách tối ưu khác nhau

Bài sau ta sẽ bắt đầu đến với việc tối ưu truy vấn ngắn
