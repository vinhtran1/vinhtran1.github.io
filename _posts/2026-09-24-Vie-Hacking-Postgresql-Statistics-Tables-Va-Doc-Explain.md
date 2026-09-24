---
title: 'Hacking PostgreSQL Statistics Tables và Bí thuật đọc hiểu EXPLAIN ANALYZE'
date: 2026-09-24 14:00:00 +0700
categories: [Database, PostgreSQL]
tags: [PostgreSQL, Query Optimization, EXPLAIN ANALYZE, pg_stats, Database Internals, SQL Tuning]
keywords: [PostgreSQL, Query Optimization, EXPLAIN ANALYZE, pg_stats, Database Internals]
pin: false
image:
  path: /assets/img/posts/2026/hacking-postgresql-statistics-tables-va-doc-explain/cover.webp
  alt: 'Hacking PostgreSQL Statistics Tables pg_statistic pg_stats và kỹ thuật đọc EXPLAIN ANALYZE'
---

# I. Dẫn nhập

Chào các bạn, có một tình huống kinh điển mà gần như bất cứ ai làm việc với PostgreSQL cũng từng ít nhất một lần vò đầu bứt tai thốt lên: *"Tại sao PostgreSQL lại khùng thế này?!"*.

Đó là khi bạn đã tạo Index đầy đủ, câu lệnh SQL viết rất chỉn chu, bảng dữ liệu có tới 10 triệu dòng, nhưng khi chạy thực tế thì PostgreSQL kiên quyết từ chối sử dụng Index Scan mà lại đè bảng ra chạy `Seq Scan` (Sequential Scan), làm CPU máy chủ chạm đỉnh 100% trong suốt 10 giây!

Phản ứng tự nhiên của nhiều bạn là tìm cách "ép" database dùng index (ví dụ như lệnh `SET enable_seqscan = off;` trong session). Nhưng cách chữa cháy đó giống như việc bạn uống thuốc giảm đau mà không chịu tìm nguyên nhân gây bệnh: **PostgreSQL Cost-Based Optimizer (CBO) không bao giờ hành động ngẫu nhiên hay vô lý!**

Bộ não lập kế hoạch (Query Planner) của PostgreSQL quyết định chọn `Seq Scan` hay `Index Scan` hoàn toàn dựa trên các phép toán xác suất và số liệu thống kê được lưu giữ trong các bảng catalog hệ thống: **`pg_statistic`** (bảng nhị phân) và view thân thiện **`pg_stats`**.

Nếu số liệu thống kê bị cũ (stale stats), hoặc dữ liệu có độ lệch phân bố quá lớn (data skew), hoặc có sự phụ thuộc lẫn nhau giữa nhiều cột (cạm bẫy Independence Assumption), Planner sẽ tính toán ra một con số chi phí (cost) hoàn toàn sai lệch so với thực tế và chọn nhầm một kế hoạch thực thi thảm họa.

Trong bài viết này, mình sẽ cùng các bạn "hack" vào bộ não của PostgreSQL: giải mã từng chỉ số vàng trong `pg_stats` (`null_frac`, `n_distinct`, `most_common_vals`, `histogram_bounds`, `correlation`), chỉ ra cách khắc phục lỗi ước lượng bằng Extended Statistics, và trang bị phương pháp 4 bước đọc `EXPLAIN (ANALYZE, BUFFERS)` chuẩn chuyên gia để bắt đúng bệnh cho mọi câu truy vấn chậm!

---

# II. Kiến trúc / Nguyên lý

### 1. Hành trình một câu truy vấn trong lõi PostgreSQL

Khi các bạn gửi một chuỗi SQL `SELECT * FROM orders WHERE ...` đến PostgreSQL, nó không được thực thi ngay mà phải trải qua một pipeline gồm 4 trạm kiểm soát:

```
[ SQL String ] ──► [ 1. Parser ] ──► [ 2. Rewriter ] ──► [ 3. Planner / Optimizer ] ──► [ 4. Executor ]
                     (Phân tích cú     (Áp dụng Rules,       (Tính toán Cost dựa trên      (Đọc dữ liệu,
                      pháp & AST)       Views, RLS)           số liệu thống kê pg_stats)    trả kết quả)
```

Trạm quan trọng nhất quyết định tốc độ chính là **Planner / Optimizer**:
- Planner tạo ra hàng chục phương án thực thi khả dĩ (Paths): Dùng Index A hay Index B? Dùng Nested Loop, Hash Join hay Merge Join?
- Đối với mỗi phương án, Planner tính toán một giá trị gọi là **Cost** (chi phí trừu tượng đo bằng đơn vị I/O đọc 1 page tuần tự `seq_page_cost = 1.0`).
- **Phương án nào có Total Cost thấp nhất sẽ được chọn làm Execution Plan chính thức để trao cho Executor!**

Để tính toán được Cost, Planner bắt buộc phải dự đoán: *"Câu query này sẽ lọc ra bao nhiêu dòng (rows)?"*. Và câu trả lời nằm toàn bộ trong **`pg_stats`**.

---

### 2. Giải mã 5 chỉ số vàng trong View `pg_stats`

View `pg_stats` cho phép chúng ta nhìn thấy trực quan dữ liệu thống kê mà tiến trình ngầm `ANALYZE` (hoặc Autovacuum worker) đã thu thập được từ một mẫu ngẫu nhiên (mặc định lấy mẫu 30,000 dòng):

```
+-----------------------------------------------------------------------------------------------+
|                               5 CHỈ SỐ VÀNG TRONG PG_STATS                                    |
+-------------------+---------------------------------------------------------------------------+
| Tên cột           | Ý nghĩa và tác động trực tiếp tới Planner                                 |
+-------------------+---------------------------------------------------------------------------+
| null_frac         | Tỉ lệ phần trăm giá trị NULL (từ 0.0 đến 1.0). Quyết định việc dùng     |
|                   | Partial Index hoặc bỏ qua điều kiện IS NULL.                             |
| n_distinct        | Số lượng giá trị phân biệt. Số dương = số đếm cố định;                     |
|                   | Số âm (-0.2) = tỉ lệ 20% tổng số dòng bảng; -1.0 = Unique 100%.          |
| most_common_vals  | Mảng chứa các giá trị xuất hiện nhiều nhất (MCV).                         |
| most_common_freqs | Tần suất xuất hiện tương ứng của các phần tử trong MCV.                    |
| histogram_bounds  | Biểu đồ tần suất chia đều (Equal-depth Histogram) cho các giá trị        |
|                   | nằm ngoài MCV, dùng để ước lượng các khoảng range (BETWEEN, <, >).         |
| correlation       | Hệ số tương quan vật lý (-1.0 đến +1.0) giữa thứ tự logic và vị trí        |
|                   | sắp xếp của các block trên ổ đĩa.                                         |
+-------------------+---------------------------------------------------------------------------+
```

Hãy đào sâu vào hai chỉ số thú vị nhất:

#### A. MCV (Most Common Values) & MCF (Most Common Frequencies)
Giả sử các bạn có bảng `users` với cột `country_code`. 80% người dùng đến từ Việt Nam (`VN`), 10% đến từ Mỹ (`US`), và 10% còn lại chia đều cho 50 quốc gia khác.
- Nếu query `WHERE country_code = 'VN'`: Planner nhìn vào MCV và biết ngay sẽ có khoảng 80% số dòng của bảng thỏa mãn. Với tỉ lệ lớn như vậy, việc dùng Index Scan sẽ gây Random I/O rất lớn -> Planner thông minh chọn **Seq Scan**.
- Nếu query `WHERE country_code = 'SG'`: Planner thấy `SG` không nằm trong MCV, nó tra cứu histogram và ước lượng chỉ có 0.2% số dòng thỏa mãn -> Planner lập tức kích hoạt **Index Scan**!

#### B. Correlation (Hệ số tương quan vật lý)
Đây là chỉ số "sống còn" quyết định giữa **Index Scan**, **Bitmap Heap Scan**, và **Seq Scan**:
- **`correlation` gần 1.0 (hoặc -1.0):** Dữ liệu được sắp xếp vật lý trên đĩa trùng khớp hoàn toàn với thứ tự của Index. Khi đọc Index, đầu đọc đĩa chỉ việc quét tuần tự các block liên tiếp (Sequential Read). Index Scan đạt tốc độ tối đa!
- **`correlation` xấp xỉ 0.0:** Dữ liệu bị phân tán rải rác khắp ổ đĩa. Một câu query lấy 5,000 dòng có thể buộc đĩa phải nhảy ngẫu nhiên tới 5,000 block khác nhau (Random I/O). Trong trường hợp này, Planner sẽ chuyển sang dùng **Bitmap Index Scan** (tạo bitmap trong RAM để gom các block cùng trang rồi mới đọc đĩa) hoặc chuyển hẳn sang **Seq Scan** nếu bảng nằm vừa trong cache!

---

### 3. Cạm bẫy "Độc lập xác suất" (Independence Assumption)

Đây là nguyên nhân phổ biến nhất khiến Planner đưa ra ước lượng sai lệch hàng ngàn lần trong thực tế.

Theo mặc định, khi gặp nhiều điều kiện trong mệnh đề `WHERE`, PostgreSQL luôn giả định các cột **độc lập thống kê với nhau**:
$$P(A \cap B) = P(A) \times P(B)$$

**Ví dụ thực tế:**  
Một bảng ô tô có cột hãng xe `make` và dòng xe `model`.
- Tỉ lệ xe `make = 'Porsche'` là $1\%$ ($0.01$).
- Tỉ lệ xe `model = '911'` là $1\%$ ($0.01$).
- Khi bạn query: `WHERE make = 'Porsche' AND model = '911'`:
  Planner nhân hai xác suất: $0.01 \times 0.01 = 0.0001$ ($0.01\%$). Với bảng 1 triệu dòng, Planner tính toán rằng chỉ có **100 dòng** thỏa mãn!
- Nhưng trên thực tế, **đã là model 911 thì chắc chắn 100% hãng xe phải là Porsche!** Số lượng dòng thực tế là **10,000 dòng** (lệch gấp 100 lần so với dự đoán của Planner).
- Hậu quả: Vì nghĩ chỉ có 100 dòng, Planner chọn `Nested Loop Join` thay vì `Hash Join`. Kết quả là truy vấn chạy mất 10 giây thay vì 20 mili-giây!

Giải pháp của PostgreSQL từ phiên bản 10 trở lên là: **Extended Statistics (`CREATE STATISTICS`)**.

---

# III. Cài đặt / Hands-on code

### Thực nghiệm 1: Soi trực tiếp bảng `pg_stats` trên dữ liệu thực tế

Hãy tạo một bảng đơn hàng và kiểm tra các thông số thống kê mà engine thu thập:

```sql
DROP TABLE IF EXISTS orders CASCADE;
CREATE TABLE orders (
    order_id BIGSERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    status VARCHAR(20) NOT NULL,
    total_amount NUMERIC(10,2),
    created_at TIMESTAMPTZ NOT NULL
);

-- Sinh 1,000,000 đơn hàng với dữ liệu lệch phân bố (Skewed Data)
INSERT INTO orders (customer_id, status, total_amount, created_at)
SELECT 
    (random()*50000)::INT,
    -- 70% COMPLETED, 20% PENDING, 8% CANCELLED, 2% REFUNDED
    (CASE 
        WHEN random() < 0.70 THEN 'COMPLETED'
        WHEN random() < 0.90 THEN 'PENDING'
        WHEN random() < 0.98 THEN 'CANCELLED'
        ELSE 'REFUNDED'
     END),
    (random()*500 + 10)::NUMERIC(10,2),
    '2026-01-01 00:00:00+07'::TIMESTAMPTZ + (g || ' seconds')::INTERVAL
FROM generate_series(1, 1000000) AS g;

-- Kích hoạt tiến trình thu thập số liệu thống kê
ANALYZE orders;
```

Bây giờ hãy soi vào view `pg_stats`:
```sql
SELECT 
    attname AS column_name,
    null_frac,
    n_distinct,
    correlation,
    most_common_vals::text AS mcv,
    most_common_freqs::text AS mcf
FROM pg_stats
WHERE tablename = 'orders' AND attname IN ('status', 'created_at');
```

**Kết quả từ database:**
```text
 column_name | null_frac | n_distinct | correlation |                  mcv                  |              mcf              
-------------+-----------+------------+-------------+---------------------------------------+-------------------------------
 status      |         0 |          4 |   0.0012582 | {COMPLETED,PENDING,CANCELLED,REFUNDED}| {0.6998,0.2001,0.0802,0.0199}
 created_at  |         0 |         -1 |           1 |                                       | 
```
Nhìn vào kết quả:
- Cột `status`: Có đúng 4 giá trị phân biệt. Tần suất MCV phản ánh chính xác 70% `COMPLETED` và 2% `REFUNDED`. Hệ số `correlation` gần bằng 0 cho thấy các trạng thái xuất hiện ngẫu nhiên.
- Cột `created_at`: `n_distinct = -1` (mỗi dòng là một giá trị duy nhất), `correlation = 1` (các dòng được ghi nối tiếp tăng dần hoàn hảo trên đĩa vật lý).

---

### Thực nghiệm 2: Cứu câu query bị Planner ước lượng sai bằng `CREATE STATISTICS`

Chúng ta tái hiện chính xác lỗi độc lập xác suất với bài toán hãng xe:

```sql
DROP TABLE IF EXISTS car_inventory CASCADE;
CREATE TABLE car_inventory (
    id SERIAL PRIMARY KEY,
    make TEXT NOT NULL,
    model TEXT NOT NULL,
    vin VARCHAR(17) NOT NULL
);

-- Chèn dữ liệu có tính tương quan phụ thuộc tuyệt đối giữa Make và Model
INSERT INTO car_inventory (make, model, vin)
SELECT 'Porsche', '911', md5(g::text) FROM generate_series(1, 15000) AS g;

INSERT INTO car_inventory (make, model, vin)
SELECT 'Toyota', 'Camry', md5(g::text) FROM generate_series(1, 250000) AS g;

INSERT INTO car_inventory (make, model, vin)
SELECT 'Ford', 'F-150', md5(g::text) FROM generate_series(1, 500000) AS g;

-- Đánh index trên 2 cột
CREATE INDEX idx_car_make_model ON car_inventory (make, model);
ANALYZE car_inventory;
```

Bây giờ hãy chạy thử câu truy vấn tìm xe Porsche 911 và xem số dòng mà Planner dự đoán:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM car_inventory 
WHERE make = 'Porsche' AND model = '911';
```

**Kế hoạch thực thi trước khi có Extended Statistics:**
```text
Bitmap Heap Scan on car_inventory  (cost=6.45..112.50 rows=288 width=37) (actual time=0.840..4.120 rows=15000 loops=1)
  Recheck Cond: ((make = 'Porsche'::text) AND (model = '911'::text))
  Buffers: shared hit=185
  ->  Bitmap Index Scan on idx_car_make_model  (cost=0.00..6.38 rows=288 width=0) (actual time=0.710..0.710 rows=15000 loops=1)
        Index Cond: ((make = 'Porsche'::text) AND (model = '911'::text))
Planning Time: 0.185 ms
Execution Time: 4.820 ms
```

Hãy nhìn vào sự sai lệch:
- Planner ước lượng: **`rows=288`**
- Thực tế chạy ra: **`actual rows=15000`**!
- **Lệch hơn 52 lần!** Nếu câu query này được JOIN với 3 bảng khác, Planner sẽ chọn thuật toán `Nested Loop` với chi phí quét 15,000 vòng lặp thay vì chỉ 288 vòng lặp, khiến hệ thống tê liệt!

#### Giải cứu bằng `CREATE STATISTICS`

Chúng ta tạo thống kê đa cột mở rộng gồm hai loại: `dependencies` (độ phụ thuộc hàm) và `mcv` (bảng tần suất đa cột kết hợp):

```sql
-- Tạo thống kê mở rộng trên cặp cột make và model
CREATE STATISTICS stat_car_make_model (dependencies, mcv) 
ON make, model FROM car_inventory;

-- Chạy lại ANALYZE để engine thu thập thống kê đa cột
ANALYZE car_inventory;
```

Bây giờ hãy chạy lại chính xác câu query trên:
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM car_inventory 
WHERE make = 'Porsche' AND model = '911';
```

**Kế hoạch thực thi sau khi đã có Extended Statistics:**
```text
Bitmap Heap Scan on car_inventory  (cost=142.10..1280.50 rows=15000 width=37) (actual time=0.912..4.210 rows=15000 loops=1)
  Recheck Cond: ((make = 'Porsche'::text) AND (model = '911'::text))
  Buffers: shared hit=185
  ->  Bitmap Index Scan on idx_car_make_model  (cost=0.00..138.35 rows=15000 width=0) (actual time=0.745..0.745 rows=15000 loops=1)
        Index Cond: ((make = 'Porsche'::text) AND (model = '911'::text))
Planning Time: 0.240 ms
Execution Time: 4.890 ms
```

Quan sát dòng đầu tiên: **`rows=15000` so với `actual rows=15000`** — Planner dự đoán **chính xác 100%**! Mọi phép tính toán JOIN và bộ nhớ `work_mem` ở các tầng tiếp theo sẽ được tối ưu hoàn hảo.

---

### Thực nghiệm 3: Quy trình 4 bước đọc `EXPLAIN (ANALYZE, BUFFERS)`

Khi phân tích một câu query chậm trên production, đừng bao giờ chạy `EXPLAIN` trần trụi. Hãy luôn dùng cú pháp đầy đủ:
```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING ON) SELECT ...
```

Dưới đây là phương pháp 4 bước mình luôn áp dụng khi xử lý sự cố database:

```
+-----------------------------------------------------------------------------------------+
|                    QUY TRÌNH 4 BƯỚC ĐỌC HIỂU EXPLAIN ANALYZE                            |
+-----------------------------------------------------------------------------------------+
| Bước 1: Đọc từ trong ra ngoài, từ dưới lên trên (Bottom-Up) theo thụt đầu dòng.        |
| Bước 2: So sánh rows (Planner đoán) và actual rows (thực tế chạy).                      |
|         -> Lệch > 10x là dấu hiệu cảnh báo thống kê bị mù/lệch!                         |
| Bước 3: Đọc dòng Buffers: shared hit (từ RAM) vs read (từ đĩa vật lý).                 |
| Bước 4: Soi chiến lược Join: Nested Loop (ít dòng), Hash Join (bảng lớn),              |
|         Merge Join (dữ liệu đã sort).                                                   |
+-----------------------------------------------------------------------------------------+
```

1. **Bước 1 — Bottom-Up:** Nút thụt lề sâu nhất chính là nơi công việc bắt đầu thực hiện đầu tiên (ví dụ quét index hoặc seq scan), sau đó dữ liệu được đẩy dần lên nút cha bên ngoài (Filter, Aggregate, Sort).
2. **Bước 2 — Độ lệch dòng (Estimation Ratio):** Lấy `actual rows / rows`. Nếu con số này lệch trên 10 lần, đó là lý do chính khiến Planner chọn nhầm thuật toán Join hoặc chọn nhầm Index.
3. **Bước 3 — Phân tích I/O qua Buffers:**
   - `shared hit=1000`: Đọc 1,000 blocks từ bộ nhớ RAM `shared_buffers` (cực nhanh, micro-giây).
   - `shared read=5000`: Đọc 5,000 blocks từ ổ đĩa vật lý (chậm, mili-giây).
   - Nếu `shared read` quá lớn, các bạn cần tăng kích thước RAM hoặc tối ưu lại index để tránh quét rác.
4. **Bước 4 — Kiểm tra thao tác Sort và Hash:**
   - Nếu thấy dòng: `Sort Method: external merge  Disk: 18432kB`: Báo động đỏ! Câu query đã bị tràn bộ nhớ `work_mem` và phải ghi file tạm xuống đĩa cứng để sắp xếp. Hãy tăng `work_mem` trong session cho câu query đó (`SET work_mem = '64MB';`).

---

# IV. Lesson learned / Tổng kết

Tối ưu hóa câu truy vấn không phải là trò chơi "đoán mò" hay đặt cược may rủi. Dưới đây là 4 bài học thực chiến giúp các bạn luôn làm chủ hiệu năng cơ sở dữ liệu:

1. **`ANALYZE` là việc đầu tiên phải làm khi query chậm:**  
   Trước khi viết lại câu query hay tạo thêm index mới, hãy chạy `ANALYZE table_name;`. Rất nhiều trường hợp dữ liệu vừa được chèn hoặc xóa hàng loạt khiến số liệu thống kê trong `pg_stats` bị lỗi thời, khiến Planner đưa ra quyết định sai lầm.

2. **Tăng Statistics Target cho các cột nghiệp vụ trọng điểm:**  
   Mặc định PostgreSQL chỉ lấy mẫu 100 buckets (`default_statistics_target = 100`). Nếu một cột có hàng trăm triệu dòng với độ phân bố cực kỳ phức tạp (như mã bưu chính, danh mục sản phẩm, user tag), hãy tăng độ chi tiết mẫu thống kê lên:
   ```sql
   ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 500;
   ANALYZE orders (customer_id);
   ```

3. **Luôn dùng `CREATE STATISTICS` cho các cột có quan hệ phụ thuộc:**  
   Bất cứ khi nào bạn viết các câu query lọc trên nhiều cột có mối liên hệ logic với nhau (Quận/Huyện và Tỉnh/Thành phố, Hãng xe và Dòng xe, Năm sinh và Độ tuổi), hãy nhớ tạo Extended Statistics để triệt tiêu cạm bẫy Independence Assumption.

4. **Khai tử thói quen phán đoán hiệu năng bằng mắt thường:**  
   Một câu query trông có vẻ "ngắn và đẹp" trên editor chưa chắc đã chạy nhanh dưới database engine. Hãy luôn bắt buộc kiểm tra bằng `EXPLAIN (ANALYZE, BUFFERS)` để nhìn thấy số lượng blocks đọc từ đĩa và chi phí thực sự của từng node xử lý!
