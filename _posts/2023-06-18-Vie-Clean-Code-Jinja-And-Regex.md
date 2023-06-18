---
title: 'Clean Code: Đọc query từ file .sql sử dụng Python: Jinja template + Regex'
date: 2023-06-18 20:00:00 +0700
categories: [Clean Code]
tags: [Python, RegEx, Jinja]
pin: false
---

## 1. Dẫn nhập

Trong quá trình tham gia dự án, có trường hợp phải sử dụng code Python để thực hiện query được viết bằng SQL. Ví dụ chọn ra các dòng dữ liệu của bảng hóa đơn mua hàng (order_table) dựa theo ngày mua hàng. Vậy đoạn code sẽ có dạng như sau:

```python
query = f''' 
            SELECT *
            FROM order_table
            WHERE order_date = {order_date_to_filter}
				 '''
database_connection.execute(query)
```

Nhìn thì cách viết này vẫn ổn cho tới khi câu query đơn giản này tiến hóa trở thành một câu query dài 50 dòng, thậm chí 100 dòng. Lúc này file Python sẽ trở nên rất dài mà phần lớn chỉ là code SQL, như vậy nhìn không được “sạch” cho lắm

Trong bài viết với chủ đề Clean Code này, mình sẽ đưa ra một gợi ý để tổ chức lại code SQL và Python tốt hơn bằng cách dùng Regular Expression và Jinja template

Có thể tóm tắt kỹ thuật này như sau:

- Đọc toàn bộ nội dung file .sql
- Dùng Regular Expression để lấy ra được đoạn SQL cần sử dụng
- Dùng thư viện Jinja để truyền tham số vào thành một câu SQL hoàn chỉnh

Ok bắt đầu thôi

## 2. Cài đặt

Đầu tiên ta sẽ cài đặt thư viện Python có tên Jinja2 

```bash
pip install Jinja2==3.1.2
```

Jinja là một thư viện được biết đến nhiều nhất với khả năng sinh điền vào một template tự động với những dữ liệu truyền vào. Ở đây ta có thể xem câu query là một template, và các thông tin ở mệnh đề WHERE và FROM chính là những thông tin ta cần truyền vào

## 3. Viết code SQL

Giả sử một file **queries.sql** có chứa tổng cộng 3 đoạn SQL với tổng độ dài hơn 100 dòng. Để dễ đọc, dễ phân tách các đoạn query thì file **queries.sql** có cấu trúc như sau:

```sql
----------------------------------------BEGIN SQL QUERY----------------------------------------
--------------------NAME: GET ORDER DATA BY ORDER DATE--------------------
SELECT * 
FROM order
WHERE order_date = {% raw %}{% for od_date in list_order_dates %}'{{ od_date }}' {% if not loop.last %} 
OR order_date = {% endif %}{% endfor %}{% endraw %};
----------------------------------------END SQL QUERY----------------------------------------

----------------------------------------BEGIN SQL QUERY----------------------------------------
--------------------NAME: ANY NAME HERE 2--------------------
--50 lines of code here
SELECT * 
FROM some_others_tables
WHERE column_name_1 = 'value1';
----------------------------------------END SQL QUERY----------------------------------------

----------------------------------------BEGIN SQL QUERY----------------------------------------
--------------------NAME: ANY NAME HERE 3--------------------
--50 lines of code here
SELECT * 
FROM some_other_tables;
----------------------------------------END SQL QUERY----------------------------------------
```

Mỗi câu query được kẹp bằng cặp văn bản **BEGIN SQL QUERY** và **END SQL QUERY** để có thể dễ đọc. Bên cạnh đó có một dòng để đặt tên cho câu query đó. Ta cần đặt một cái tên gọn, nhưng đầy đủ ý nghĩa, đọc lên là có thể hình dung ra nội dung câu query đó có chức năng gì, và đặc biệt là nó không được trùng với các tên khác có trong cùng file **queries.sql**

**Ghi chú**: Đoạn SQL có tên **GET ORDER DATA BY ORDER DATE** đã được viết theo dạng template mà thư viện Jinja có thể hiểu được. Ý nghĩa của đoạn code này là nếu ta truyền vào danh sách gồm 1 order date, ví dụ **[‘2023-06-01’]**, thì câu query sẽ có dạng là

```sql
SELECT * 
FROM order
WHERE order_date = '2023-01-01'
```

Nếu ta truyền vào một danh sách gồm nhiều order date, ví dụ **[’2023-06-01’, ‘2023-06-02’, ‘2023-06-03’]** thì câu query sẽ có dạng là:

```sql
SELECT * 
FROM order
WHERE order_date = '2023-01-01'
OR order_date = '2023-01-02'
OR order_date = '2023-01-03'
```

Như vậy là ta đã đem code SQL lưu trữ trong file .sql. Bước tiếp theo ta sẽ viết code Python để đọc và hoàn thiện câu query

## 4. Viết code Python

### 4.1 Đọc file SQL

Đầu tiên ta cần đọc toàn bộ nội dung file **queries.sql**

```python
with open('queries.sql', 'r') as f:
    all_queries = f.read()
```

### 4.2 Viết regex

Nói sơ cho ai chưa biết về regex. Đây là một chuỗi ký tự được gọi là pattern. Chức năng chính của nó sẽ đi tìm kiếm trong một đoạn văn bản những văn bản nhỏ hơn mà tuân theo pattern đó. 

Ví dụ tìm những văn bản có thể là đại diện cho một email hợp lệ trong một đoạn văn lớn sẽ cần dùng đến Regular Expression

Quay trở lại đề bài. Đầu tiên ta định nghĩa một pattern regex như sau:

```python
RE_GET_SQL = '(?<=-{{40}}BEGIN SQL QUERY-{{40}}\n-{{20}}NAME: {sql_query_name}-{{20}}\n)(?:(?!-{{40}}END SQL QUERY-{{40}})[\s\S])*(?=-{{40}}END SQL QUERY-{{40}})'
```

Sau đó truyền vào tên của câu SQL cần lấy ra

```python
sql_query_name = 'GET ORDER DATA BY ORDER DATE'
re_pattern = RE_GET_SQL.format(sql_query_name=sql_query_name)
print(re_pattern)
```

Kết quả của regex sẽ là:

```
(?<=-{40}BEGIN SQL QUERY-{40}
-{20}NAME: GET ORDER DATA BY ORDER DATE-{20}
)(?:(?!-{40}END SQL QUERY-{40})[\s\S])*(?=-{40}END SQL QUERY-{40})
```

Đoạn regex này có thể dịch ra là: Tìm những đoạn văn bản được kẹp bởi  **BEGIN SQL QUERY** và **END SQL QUERY** và có tên là **GET ORDER DATA BY ORDER DATE**

Để test đoạn regex này, ta có thể copy kết quả lên trang web [https://regex101.com/](https://regex101.com/)

![RegexMatched](https://thumbs2.imgbox.com/df/17/1U17ubIB_t.png)

Như trong hình có thể thấy, đoạn SQL có tên **GET ORDER DATA BY ORDER DATE** đã được tìm thấy (đoạn được tô màu xanh). 

### 4.3 Lấy SQL theo tên

Đoạn code Python này dùng để lấy đúng đoạn SQL theo tên đã truyền vào. Nếu không tìm thấy thì trả về giá trị mặc định

```python
import re
match = re.search(re_pattern, all_queries)
try:
    query_template = match.group()
    print(query_template)
except:
    query_template = "No query template"
    print("No query template matched")
```

Như vậy là ta đã lấy được đoạn SQL cần sử dụng theo tên. Tiếp theo ta sẽ sử dụng Jinja template để truyền vào các tham số để có được đoạn SQL sau cùng

### 4.4 Dùng Jinja để truyền vào các giá trị để hoàn chỉnh câu query

```python
import jinja2
jinja_template = jinja2.Template(query_template)
param = {'list_order_dates': ['2023-01-01', '2020-01-02', '2020-01-04']}
query = jinja_template.render(param)
print(query)
```

Kết quả hiển thị:

```sql
--30 lines of code here
SELECT * 
FROM order
WHERE order_date = '2023-01-01'  
OR order_date = '2020-01-02'  
OR order_date = '2020-01-04' ;
```

Vậy là đã có được một câu query có thể đem đi thực thi

Bây giờ ta sẽ ghép code lại thôi

### 4.5 Code tổng hợp

Tổng hợp lại các đoạn code ở trên, ta có được file Python như sau

```python
# filename sql_reader.py
import re
import jinja2
with open('queries.sql', 'r') as f:
    all_queries = f.read()

RE_GET_SQL = '(?<=-{{40}}BEGIN SQL QUERY-{{40}}\n-{{20}}NAME: {sql_query_name}-{{20}}\n)(?:(?!-{{40}}END SQL QUERY-{{40}})[\s\S])*(?=-{{40}}END SQL QUERY-{{40}})'
sql_query_name = 'GET ORDER DATA BY ORDER DATE'
re_pattern = RE_GET_SQL.format(sql_query_name=sql_query_name)
print(re_pattern)

match = re.search(re_pattern, all_queries)
try:
    query_template = match.group()
    print(query_template)
except:
    query_template = "No query template"
    print("No query template matched")

jinja_template = jinja2.Template(query_template)
param = {'list_order_dates': ['2023-01-01', '2020-01-02', '2020-01-04']}
query = jinja_template.render(param)
print(query)

# Execute query
# database_connection.execute(query)
```
Kết quả sau khi chạy file Python sẽ là:

```
(?<=-{40}BEGIN SQL QUERY-{40}
-{20}NAME: GET ORDER DATA BY ORDER DATE-{20}
)(?:(?!-{40}END SQL QUERY-{40})[\s\S])*(?=-{40}END SQL QUERY-{40})
--------------------------------
SELECT * 
FROM order
WHERE order_date = {% raw %}{% for od_date in list_order_dates %}'{{ od_date }}' {% if not loop.last %} 
OR order_date = {% endif %}{% endfor %}{% endraw %};

--------------------------------
SELECT * 
FROM order
WHERE order_date = '2023-01-01'  
OR order_date = '2020-01-02'  
OR order_date = '2020-01-04' ;
--------------------------------
```
Đến đây xem như hoàn thành rồi. Bây giờ thì code đã gọn gàng, dễ đọc hơn rồi vì SQL và Python lúc này nằm tách biệt với nhau.

## 5. Kết luận

Hy vọng với một kỹ thuật nhỏ này sẽ giúp cho việc viết code thêm chuyên nghiệp hơn, “sạch” hơn

Kỹ thuật này có thể áp dụng để Python đọc một số loại text có format khác với Python chứ không chỉ riêng SQL

Mọi người có thể lấy code đầy đủ tại link Github ở đây nhé: [Link](https://github.com/vinhtran1/jinja_and_regex)