---
title: 'SQL Optimization - Bài 1: Tối ưu truy vấn ngắn bằng Index'
date: 2026-09-24 09:15:00 +0700
categories: [Theory, Database]
tags: [SQL, PostgreSQL, Index, Optimization, Performance Tuning, Index Selectivity, Functional Index, Bitmap Heap Scan]
keywords: [Index Selectivity, Functional Index, Bitmap Heap Scan, PostgreSQL, Index]
pin: false
image:
  path: /assets/img/posts/2026/sql-optimization-p1/cover.webp
  alt: 'SQL Optimization: Tối ưu truy vấn ngắn bằng Index trong PostgreSQL'
---

# I. Dẫn nhập

Chào các bạn, đây là bài viết tiếp theo trong chuỗi series tối ưu hóa SQL (SQL Optimization) mà mình muốn chia sẻ. Khi bắt đầu học và làm việc với cơ sở dữ liệu quan hệ, việc viết được một câu lệnh SQL trả về đúng và đủ dữ liệu cho nghiệp vụ là bước đầu tiên. Tuy nhiên, khi hệ thống đi vào môi trường sản xuất (production) với hàng triệu hay hàng chục triệu bản ghi, việc câu SQL đó mất 5 mili-giây hay 5 giây để thực thi sẽ quyết định sự sống còn của toàn bộ ứng dụng.

Trong bài viết này, mình sẽ tập trung vào kỹ thuật tối ưu hóa cho **truy vấn ngắn (Short Queries)**.

> **Truy vấn ngắn (Short Queries)** là các truy vấn chỉ trích xuất hoặc tính toán trên một tập con dữ liệu rất nhỏ từ các bảng thành phần, thông thường là **dưới 10%**, và trong nhiều trường hợp thực tế là **dưới 1%** tổng số dòng của bảng.

Trong các hệ thống xử lý giao dịch trực tuyến (OLTP - Online Transaction Processing) như ứng dụng thương mại điện tử, mạng xã hội, ngân hàng hay fintech, hơn 90% các request từ người dùng thuộc về nhóm truy vấn ngắn này: Đăng nhập tài khoản, xem chi tiết một đơn hàng, lấy danh sách 10 thông báo mới nhất. Và vũ khí tối thượng, nền tảng số một để tối ưu truy vấn ngắn không gì khác chính là **Index**.

Tuy nhiên, liệu có phải cứ tạo Index là câu truy vấn sẽ tự động chạy nhanh như bay? Tại sao nhiều lúc bạn đã đánh Index cẩn thận nhưng khi kiểm tra kế hoạch thực thi (Execution Plan), cơ sở dữ liệu vẫn lẳng lặng chạy quét toàn bộ bảng (**Sequential Scan**)? Hãy cùng mình đi sâu vào kiến trúc và nguyên lý cốt lõi ngay dưới đây.

---

# II. Kiến trúc / Nguyên lý

### 1. Nguyên lý Index Selectivity (Độ chọn lọc của Index)

Để hiểu tại sao Index hoạt động hiệu quả, chúng ta phải nắm rõ khái niệm **Index Selectivity (Độ chọn lọc)**.

> **Selectivity** phản ánh tỷ lệ giữa số lượng giá trị duy nhất (Cardinality) trên tổng số dòng của bảng.
>
> $$Selectivity = \frac{\text{Số lượng giá trị duy nhất (Distinct Values)}}{\text{Tổng số dòng trong bảng (Total Rows)}}$$

- **Selectivity cao (tiến gần đến 1.0):** Cột có tính duy nhất rất lớn. Ví dụ: `user_id`, `email`, `uuid`. Khi tìm kiếm theo một giá trị cụ thể, câu truy vấn chỉ trỏ tới duy nhất 1 dòng (hoặc một vài dòng). Đây là môi trường lý tưởng nhất để B-tree Index phát huy sức mạnh vượt trội ($O(\log N)$).
- **Selectivity thấp (tiến gần đến 0):** Cột có rất ít giá trị phân biệt. Ví dụ: cột `gender` chỉ có `M/F`, cột `status` chỉ có `ACTIVE/INACTIVE`, hoặc một cột 10 triệu dòng nhưng chỉ có hai giá trị `0` và `1`.

Chính vì lý do này, PostgreSQL (và hầu hết các hệ quản trị CSDL quan hệ khác) sẽ tự động tạo Index ngầm định khi bạn khai báo ràng buộc **Primary Key** hoặc **Unique Constraint**, bởi vì tính duy nhất của các cột này là tuyệt đối ($Selectivity = 1.0$). 

Ngược lại, nếu bạn cố tình đánh B-tree Index trên một cột có Selectivity quá thấp, việc đọc qua Index không những không làm câu truy vấn nhanh hơn mà còn làm chậm đi, do chi phí phải đọc thêm các khối dữ liệu Index (Index Blocks) rồi mới trỏ đến các khối dữ liệu bảng thực tế (Heap Table Blocks).

### 2. Tại sao Bộ tối ưu (Optimizer) chọn Seq Scan thay vì Index Scan?

Nhiều bạn thường thắc mắc: *"Mình đã tạo B-tree Index trên cột đó rồi, tại sao khi chạy EXPLAIN thì PostgreSQL vẫn chạy Sequential Scan (Seq Scan)?"*.

Nguyên nhân nằm ở cơ chế **Cost-based Optimizer (CBO)** của cơ sở dữ liệu. Bộ tối ưu hóa không bao giờ chọn thuật toán một cách cảm tính; nó tính toán chi phí (Cost) dựa trên thống kê phân phối dữ liệu (Statistics):
- **Sequential Page Cost (Chi phí đọc tuần tự):** Ổ cứng (đặc biệt là HDD truyền thống và cả SSD với block cache) đọc tuần tự các trang dữ liệu liên tiếp với tốc độ rất cao.
- **Random Page Cost (Chi phí đọc ngẫu nhiên):** Việc tra cứu qua Index đòi hỏi con trỏ phải nhảy ngẫu nhiên liên tục giữa các trang đĩa của Index và các trang đĩa của Heap Table.

Khi độ chọn lọc giảm đi (nghĩa là tập kết quả trả về chiếm tỷ lệ lớn trong bảng, ví dụ trên 15% - 20%), chi phí đọc ngẫu nhiên qua Index sẽ vượt qua chi phí đọc tuần tự toàn bộ bảng. Lúc này, Optimizer sẽ quyết định bỏ qua Index và quét thẳng toàn bộ bảng bằng **Seq Scan**.

Thông thường, đối với PostgreSQL, tùy thuộc vào tỷ lệ dữ liệu thỏa mãn điều kiện, kế hoạch thực thi sẽ chuyển đổi nhịp nhàng qua 3 trạng thái:
1. **Index Scan:** Được chọn khi tập kết quả cực nhỏ (< 1% - 3%). Con trỏ đi trực tiếp từ B-tree Index đến từng dòng trong Heap Table.
2. **Bitmap Heap Scan / Bitmap Index Scan:** Được chọn khi tập kết quả ở mức trung bình (~5% - 25%). Optimizer quét Index để dựng một bản đồ bit (Bitmap) trong bộ nhớ, đánh dấu các trang dữ liệu cần đọc, sau đó đọc các trang đó một cách tuần tự theo thứ tự đĩa vật lý để hạn chế tối đa đọc ngẫu nhiên.
3. **Seq Scan (Sequential Scan):** Được chọn khi tập kết quả lớn (> 25% - 30%). Quét tuần tự toàn bộ bảng từ đầu đến cuối là giải pháp tối ưu nhất về mặt I/O.

### 3. Hành vi của giá trị NULL trong Index (PostgreSQL vs Oracle/MySQL)

Một điểm kiến trúc cực kỳ quan trọng mà các kỹ sư phần mềm thường bỏ sót khi chuyển đổi giữa các hệ CSDL quan hệ:
- **Trong PostgreSQL và SQL Server:** B-tree Index lưu trữ và đánh chỉ mục cho cả các giá trị **NULL**. Do đó, điều kiện `WHERE col IS NULL` hoàn toàn có thể tận dụng B-tree Index để tăng tốc.
- **Trong Oracle và MySQL InnoDB:** Mặc định, nếu tất cả các cột trong một mục Index đều là `NULL`, mục đó sẽ **KHÔNG được đưa vào B-tree Index**. Vì vậy, câu truy vấn `WHERE col IS NULL` trên Oracle hay MySQL sẽ tự động biến thành Full Table Scan nếu không có biện pháp can thiệp đặc biệt!

Để xử lý bài toán này trong môi trường đa cơ sở dữ liệu hoặc khi cần tối ưu tuyệt đối:
- **Cách 1: Tạo Partial Index (Chỉ mục một phần):**
  Trong PostgreSQL, nếu bảng có hàng triệu dòng và chỉ có một lượng rất nhỏ bản ghi bị NULL (ví dụ: các giao dịch chưa xử lý `processed_at IS NULL`), ta tạo Partial Index:
  ```sql
  CREATE INDEX idx_unprocessed_orders ON orders (order_id) WHERE processed_at IS NULL;
  ```
  Index này siêu nhỏ gọn và cực kỳ nhanh.
- **Cách 2: Sử dụng giá trị mặc định (Sentinel Values):**
  Thay thế giá trị NULL bằng một giá trị quy ước cụ thể (ví dụ: `-1` cho số nguyên, `'1900-01-01'` cho ngày tháng, `'N/A'` cho chuỗi). Phương án này giúp logic truy vấn nhất quán, tránh được các bẫy logic 3 giá trị (Three-valued logic: True, False, Unknown) khi thực hiện các phép `JOIN`.

---

# III. Cài đặt / Hands-on code

Bây giờ, chúng ta sẽ bắt tay vào phần thực nghiệm để trực tiếp quan sát hành vi của PostgreSQL Optimizer khi làm việc với Index.

### Bước 1: Thử nghiệm hiện tượng "chết" Index do hàm biến đổi cột

Giả sử ta có bảng khách hàng `test.clients` chứa 100,000 bản ghi và ta tạo một B-tree Index trên cột ngày sinh `date_of_birth`:

```sql
-- Tạo bảng mô phỏng
CREATE SCHEMA IF NOT EXISTS test;
CREATE TABLE test.clients (
    client_id SERIAL PRIMARY KEY,
    full_name VARCHAR(100),
    email VARCHAR(100),
    date_of_birth TIMESTAMP NOT NULL
);

-- Tạo Index trên cột date_of_birth
CREATE INDEX clients_date_of_birth_idx 
    ON test.clients USING btree (date_of_birth);
```

Giả sử nghiệp vụ yêu cầu: *Tìm kiếm tất cả khách hàng có độ tuổi lớn hơn 100 tuổi*.

Một lập trình viên quen tư duy toán học thông thường có thể sẽ viết câu SQL như sau: Lấy năm hiện tại trừ đi năm sinh, nếu lớn hơn 100 thì thỏa mãn:

```sql
EXPLAIN
SELECT * 
FROM test.clients c 
WHERE extract(year FROM current_date) - extract(year FROM date_of_birth) > 100;
```

Kết quả EXPLAIN trả về từ PostgreSQL sẽ khiến bạn bất ngờ:

```
QUERY PLAN                                                                                       
-------------------------------------------------------------------------------------------------
Seq Scan on clients c  (cost=0.00..4370.00 rows=33333 width=130)                                 
  Filter: ((EXTRACT(year FROM CURRENT_DATE) - EXTRACT(year FROM date_of_birth)) > '100'::numeric)
```

**Tại sao Index bị vô hiệu hóa?**
Bởi vì Index được dựng trên giá trị nguyên bản của cột `date_of_birth`. Khi bạn bọc hàm `extract(year FROM date_of_birth)`, CSDL không thể biết trước giá trị sau khi tính toán của hàm đó tương ứng với nhánh nào trên cây B-tree. Do đó, Optimizer bắt buộc phải quét tuần tự toàn bộ bảng (Seq Scan) và tính hàm `extract()` cho từng dòng một!

**Cách khắc phục chuẩn xác:**
Chuyển đổi toàn bộ biểu thức tính toán sang vế phải để giữ nguyên cột `date_of_birth` ở vế trái:

```sql
EXPLAIN  
SELECT * 
FROM test.clients c 
WHERE date_of_birth < '1926-01-01 00:00:00'::timestamp;
```

Kết quả EXPLAIN lập tức thay đổi:

```
QUERY PLAN                                                                                 
-------------------------------------------------------------------------------------------
Index Scan using clients_date_of_birth_idx on clients c  (cost=0.29..8.31 rows=1 width=130)
  Index Cond: (date_of_birth < '1926-01-01 00:00:00'::timestamp without time zone)         
```

Optimizer đã chuyển ngay sang **Index Scan** với cost giảm từ `4370.00` xuống còn vỏn vẹn `8.31`!

### Bước 2: Quan sát sự chuyển dịch kế hoạch khi thay đổi Selectivity

Bây giờ ta thay đổi điều kiện lọc để quan sát hành vi của Optimizer:
- **Lọc khách hàng trên 70 tuổi (sinh trước 1956, chiếm ~29,000 dòng):**
  ```sql
  EXPLAIN
  SELECT * FROM test.clients c 
  WHERE date_of_birth < '1956-01-01 00:00:00'::timestamp;
  ```
  Kế hoạch thực thi chuyển sang **Bitmap Heap Scan**:
  ```
  QUERY PLAN                                                                                  
  --------------------------------------------------------------------------------------------
  Bitmap Heap Scan on clients c  (cost=424.44..2912.41 rows=29438 width=130)                  
    Recheck Cond: (date_of_birth < '1956-01-01 00:00:00'::timestamp without time zone)        
    ->  Bitmap Index Scan on clients_date_of_birth_idx  (cost=0.00..417.08 rows=29438 width=0)
          Index Cond: (date_of_birth < '1956-01-01 00:00:00'::timestamp without time zone)    
  ```

- **Lọc khách hàng trên 30 tuổi (sinh trước 1996, chiếm hơn 84,000 dòng / 100,000 dòng):**
  ```sql
  EXPLAIN
  SELECT * FROM test.clients c 
  WHERE date_of_birth < '1996-01-01 00:00:00'::timestamp;
  ```
  Kế hoạch thực thi tự động quay về **Seq Scan**:
  ```
  QUERY PLAN                                                                    
  ------------------------------------------------------------------------------
  Seq Scan on clients c  (cost=0.00..3370.00 rows=84126 width=130)              
    Filter: (date_of_birth < '1996-01-01 00:00:00'::timestamp without time zone)
  ```
  Lúc này, việc đọc tuần tự mang lại throughput cao hơn hẳn so với việc tra cứu Index cho 84% dữ liệu của bảng.

### Bước 3: Ba tuyệt chiêu viết SQL để khai phóng tối đa sức mạnh của Index

Dưới đây là 3 kỹ thuật thực chiến mà mình luôn áp dụng khi review code SQL của team:

#### Kỹ thuật 1: Sử dụng Functional Index (Expression Index) cho chuỗi ký tự

Khi tìm kiếm chuỗi không phân biệt hoa thường (Case-insensitive search), lập trình viên thường viết:
```sql
SELECT * FROM test.clients WHERE lower(email) = 'john.doe@example.com';
```
Nếu chỉ có index thường trên `email`, câu lệnh trên sẽ bị Seq Scan. Giải pháp là tạo một **Functional Index** trên biểu thức `lower(email)`:

```sql
CREATE INDEX idx_clients_lower_email ON test.clients (lower(email));
```
Kể từ lúc này, mọi câu truy vấn dùng `WHERE lower(email) = ...` sẽ ăn thẳng vào `idx_clients_lower_email` với tốc độ Index Scan tuyệt đối.

#### Kỹ thuật 2: Xử lý Datetime và Timestamp bằng CTE / Subquery

Một thói quen rất phổ biến là ép kiểu cột ngày giờ về dạng date để so sánh với ngày hôm qua:
```sql
-- CÁCH VIẾT NGUY HIỂM (Mất Index của created_time)
SELECT * FROM orders 
WHERE created_time::date >= current_date - interval '1' day;
```
Biểu thức `created_time::date` đã biến đổi cột và làm mất Index. Ta nên tính toán vế phải trước bằng CTE hoặc Subquery:

```sql
-- CÁCH VIẾT TỐI ƯU (Bảo toàn Index)
WITH cte_yesterday AS (
    SELECT (current_date - interval '1' day)::timestamp AS yesterday
)
SELECT * FROM orders 
WHERE created_time >= (SELECT yesterday FROM cte_yesterday);
```
Câu truy vấn mới tuy dài hơn 2 dòng nhưng bảo toàn nguyên vẹn cột `created_time` ở vế trái, giúp câu truy vấn quét qua Index trong chớp mắt.

#### Kỹ thuật 3: Triệt tiêu hàm COALESCE trong mệnh đề WHERE

Hàm `COALESCE` cũng là một hàm biến đổi khiến Index của cả hai cột tham gia đều bị vô hiệu hóa:

```sql
-- CÁCH VIẾT NGUY HIỂM:
SELECT * FROM invoices 
WHERE coalesce(paid_date, due_date) BETWEEN '2026-08-01' AND '2026-08-31';
```

Ta có thể triệt tiêu `COALESCE` bằng cách tách thành mệnh đề logic `OR ... IS NULL`:

```sql
-- CÁCH VIẾT TỐI ƯU:
SELECT * FROM invoices 
WHERE (paid_date BETWEEN '2026-08-01' AND '2026-08-31')
   OR (paid_date IS NULL AND due_date BETWEEN '2026-08-01' AND '2026-08-31');
```

Lúc này, nếu bạn có index trên `paid_date` và index trên `due_date`, PostgreSQL sẽ sử dụng **BitmapOr** để kết hợp kết quả quét từ cả hai Index độc lập mà không cần phải duyệt qua bất kỳ dòng nào không thỏa mãn!

---

# IV. Lesson learned / Tổng kết

Tối ưu hóa truy vấn ngắn không phải là một phép màu, mà là sự hiểu biết sâu sắc về cách thức hoạt động của bộ lưu trữ và bộ tối ưu hóa truy vấn (Query Optimizer).

Để đúc kết lại bài viết này, mình gửi tới các bạn 4 bài học kinh nghiệm (Lesson Learned) cốt lõi:

1. **Hiểu rõ Index Selectivity:** Chỉ nên tạo B-tree Index trên các cột có độ chọn lọc cao (nhiều giá trị phân biệt). Tránh lãng phí tài nguyên ghi và dung lượng đĩa cho những cột chỉ có một vài trạng thái lặp đi lặp lại.
2. **Quy tắc "Bất khả xâm phạm" vế trái:** Tuyệt đối không bọc hàm (`EXTRACT`, `ROUND`, `SUBSTRING`), ép kiểu (`::date`, `CAST`) hay thực hiện phép toán số học lên cột đã có Index trong mệnh đề `WHERE` và `JOIN ... ON`. Mọi tính toán phải được chuyển dịch hoàn toàn sang vế phải.
3. **Cẩn trọng với giá trị NULL:** Luôn nhớ sự khác biệt giữa PostgreSQL (hỗ trợ Index trên NULL) và Oracle/MySQL (không lưu toàn bộ NULL trong B-tree). Tận dụng Partial Index để tối ưu các trường hợp bản ghi NULL có chọn lọc.
4. **Tận dụng Functional Index và biến đổi COALESCE:** Khi bắt buộc phải tìm kiếm theo biểu thức chuẩn hóa (như `lower()`), hãy tạo Functional Index tương ứng. Khi gặp `COALESCE`, hãy tái cấu trúc thành mệnh đề `OR ... IS NULL` để giải phóng sức mạnh của Index.

Hy vọng bài viết này giúp ích cho các bạn trong công việc tối ưu hóa cơ sở dữ liệu hàng ngày. Trong bài tiếp theo của series, mình sẽ cùng các bạn phân tích sâu hơn về Composite Index (Chỉ mục kết hợp), quy tắc Leftmost Prefix và cách giải quyết bài toán sắp xếp Sorting/Pagination hiệu năng cao!
