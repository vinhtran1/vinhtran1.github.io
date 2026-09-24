---
title: 'Cursor trong PostgreSQL và Kỹ thuật tối ưu Foreign Data Wrapper (FDW)'
date: 2026-09-24 12:00:00 +0700
categories: [Database, PostgreSQL]
tags: [PostgreSQL, Cursor, Foreign Data Wrapper, FDW, Performance Tuning, PLpgSQL]
keywords: [PostgreSQL, Cursor, Foreign Data Wrapper, FDW, Performance Tuning]
pin: false
image:
  path: /assets/img/posts/2026/postgresql-cursor-va-ung-dung-toi-uu-foreign-data-wrapper/cover.webp
  alt: 'Vòng đời Cursor trong PostgreSQL và cơ chế Fetch Chunking tối ưu Foreign Data Wrapper'
---

# I. Dẫn nhập

Chào các bạn, đã bao giờ các bạn gặp phải tình huống: Backend service (viết bằng NodeJS, Go, hay Python) thực hiện một câu lệnh truy vấn xuất báo cáo hoặc đồng bộ dữ liệu với số lượng bản ghi lên tới vài chục triệu dòng, và chỉ sau vài chục giây, toàn bộ pod container lăn đùng ra chết với tín hiệu lỗi `OOMKilled` (Out Of Memory)?

Trong thế giới phát triển ứng dụng, các lập trình viên thường có định kiến rất tiêu cực về con trỏ (**Cursor**). Chúng ta thường được dạy rằng: *"SQL là ngôn ngữ hướng tập hợp (set-based), đừng bao giờ dùng Cursor vì nó xử lý từng dòng tuần tự (row-by-row) cực kỳ chậm chạp!"*.

Lời khuyên đó không hề sai khi các bạn thực hiện các phép tính toán biến đổi logic số liệu. Nhưng khi bài toán của bạn là **xử lý khối lượng dữ liệu khổng lồ vượt quá dung lượng RAM khả dụng**, Cursor lại chính là chiếc phao cứu sinh độc nhất vô nhị giúp kiểm soát dòng chảy dữ liệu một cách an toàn và nhịp nhàng.

Thú vị hơn nữa, một trong những tính năng "thời thượng" nhất của PostgreSQL là **Foreign Data Wrapper (`postgres_fdw`)** — cho phép database của bạn truy vấn bảng dữ liệu nằm trên một cụm server PostgreSQL từ xa giống như bảng nội bộ — lại hoạt động ngầm **100% dựa trên Cursor**!

Và đây chính là nơi quả bom nổ chậm xuất hiện: Cấu hình mặc định ngầm của `postgres_fdw` là `fetch_size = 100`. Nếu các bạn truy vấn một bảng từ xa có 1 triệu dòng, database của bạn sẽ phải thực hiện tới **10,000 lượt ping-pong mạng qua lại** chỉ để kéo dữ liệu về! Trong môi trường mạng phân tán đa vùng (Cross-Region hoặc Hybrid Cloud), độ trễ mạng sẽ khiến câu query chạy mất hàng phút đồng hồ.

Trong bài viết này, mình sẽ cùng các bạn mổ xẻ toàn diện vòng đời của Cursor trong PostgreSQL, cách kết hợp Cursor để stream dữ liệu an toàn, và bí thuật tinh chỉnh `fetch_size` cùng kỹ thuật Query Pushdown trên `postgres_fdw` để tăng tốc độ truy vấn từ xa lên gấp hơn 15 lần!

---

# II. Kiến trúc / Nguyên lý

### 1. Bản chất cơ học của Cursor trong PostgreSQL

Về mặt kiến trúc, một **Cursor** (con trỏ) không phải là một bản sao dữ liệu. Nó là một **Read-only Pointer** trỏ vào kết quả đã được lên kế hoạch (Execution Plan) của một câu lệnh `SELECT`, được quản lý bên trong một cấu trúc bộ nhớ của backend gọi là **Portal**.

Thay vì buộc executor phải đọc toàn bộ các dòng dữ liệu vào bộ nhớ đệm và truyền một gói tin khổng lồ về phía client, Cursor cho phép client chủ động yêu cầu: *"Hãy đưa cho tôi đúng $N$ dòng tiếp theo!"*.

```
[ Application Client ]                 [ PostgreSQL Backend Engine ]
         │                                          │
         ├────── DECLARE cur CURSOR FOR SELECT ────►│ Tạo Portal trong bộ nhớ
         │                                          │ Lập Execution Plan (chưa chạy hết)
         │                                          │
         ├────── FETCH 5000 FROM cur ──────────────►│ Executor đọc đúng 5,000 tuples
         │◄───── Trả về 5,000 dòng ─────────────────┤ Tạm dừng tiến trình quét
         │       (Xử lý an toàn trong RAM)          │
         │                                          │
         ├────── FETCH 5000 FROM cur ──────────────►│ Executor đọc 5,000 tuples kế tiếp
         │◄───── Trả về 5,000 dòng ─────────────────┤
         │                                          │
         ├────── CLOSE cur ────────────────────────►│ Hủy Portal, giải phóng bộ nhớ
```

### 2. Vòng đời toàn diện của Cursor

Một Cursor chuẩn trong PostgreSQL trải qua 5 giai đoạn chính:

```
  +-------------+     +----------+     +----------------+     +-----------+     +-----------+
  | 1. Khai báo | --> | 2. Mở    | --> | 3. Điều hướng  | --> | 4. Cập    | --> | 5. Đóng   |
  |  (DECLARE)  |     |  (OPEN)  |     | (FETCH / MOVE) |     | nhật tại  |     |  (CLOSE)  |
  |             |     |          |     |                |     | chỗ dòng  |     |           |
  +-------------+     +----------+     +----------------+     +-----------+     +-----------+
```

1. **`DECLARE`**: Khai báo con trỏ với các thuộc tính kiểm soát hành vi:
   - `BINARY`: Đọc dữ liệu dưới định dạng nhị phân thay vì văn bản (nhanh hơn nhưng phụ thuộc kiến trúc máy).
   - `NO SCROLL` (Mặc định): Con trỏ chỉ có thể di chuyển tiến về phía trước (`FETCH NEXT / FORWARD`). Tiết kiệm bộ nhớ vì engine không cần lưu vết các dòng đã đi qua.
   - `SCROLL`: Cho phép con trỏ nhảy lùi lại phía sau (`FETCH PRIOR`), nhảy đến đầu (`FIRST`), cuối (`LAST`), hoặc nhảy tương đối (`RELATIVE n`). Engine buộc phải lưu trữ kết quả tạm thời trong một file tạm (Tuplestore).
   - `WITH HOLD` vs `WITHOUT HOLD`: Mặc định (`WITHOUT HOLD`), cursor sẽ bị tự động đóng khi transaction commit. Nếu chỉ định `WITH HOLD`, cursor vẫn tiếp tục sống sót qua các lệnh `COMMIT` cho đến khi phiên làm việc kết thúc hoặc có lệnh `CLOSE` tường minh.
2. **`OPEN`**: Gán các tham số thực tế (nếu là Bound Cursor có tham số trong PL/pgSQL) và bắt đầu kích hoạt portal.
3. **`FETCH` và `MOVE`**:
   - `FETCH n`: Kéo $n$ bản ghi tiếp theo về client.
   - `MOVE n`: Di chuyển vị trí con trỏ bỏ qua $n$ bản ghi mà không trả dữ liệu về (cực kỳ hữu ích cho việc phân trang dữ liệu sâu mà không tốn băng thông mạng).
4. **`WHERE CURRENT OF cursor_name`**:
   - Cho phép thực hiện `UPDATE` hoặc `DELETE` trực tiếp lên chính dòng mà con trỏ đang dừng lại.
   - Điều kiện bắt buộc: Câu lệnh SELECT của cursor phải có khóa `FOR UPDATE` hoặc `FOR NO KEY UPDATE`.
5. **`CLOSE`**: Giải phóng Portal, bộ nhớ `work_mem` và các chốt khóa đệm (buffer pins).

---

### 3. Mối liên kết ngầm giữa Cursor và Foreign Data Wrapper (`postgres_fdw`)

`postgres_fdw` là extension cho phép một database PostgreSQL (Local) kết nối và truy vấn các bảng nằm trên database PostgreSQL khác (Remote). 

Khi các bạn thực hiện một câu truy vấn trên foreign table:
```sql
SELECT id, user_id, amount FROM remote_orders WHERE status = 'COMPLETED';
```

Bạn có bao giờ tự hỏi: **Local database kéo dữ liệu từ Remote database về bằng cách nào?**

Dưới nắp ca-pô, PostgreSQL **không bao giờ** phát một câu lệnh SELECT thông thường để tải một cục dữ liệu khổng lồ qua socket. Thay vào đó, nó gửi chuỗi lệnh sau sang remote database:
```sql
-- Chạy trên Remote Server:
DECLARE pgfdw_cursor_0 CURSOR FOR 
SELECT id, user_id, amount FROM orders WHERE status = 'COMPLETED';

FETCH 100 FROM pgfdw_cursor_0;
FETCH 100 FROM pgfdw_cursor_0;
...
```

**Cạm bẫy độ trễ mạng (Network Ping-Pong Bottleneck):**  
Tham số mặc định điều khiển số dòng mỗi lần fetch của `postgres_fdw` là **`fetch_size = 100`**.

Hãy làm một phép tính toán học đơn giản:
- Giả sử bảng `remote_orders` trả về **1,000,000 bản ghi**.
- Với `fetch_size = 100`, Local server phải phát tới **10,000 lệnh `FETCH 100`** qua mạng Internet/VPN.
- Nếu Round-Trip Time (RTT) giữa hai server là **10 mili-giây** (khoảng cách thông thường giữa 2 vùng Cloud hoặc On-Premise kết nối lên Cloud):
  $$\text{Thời gian chết do chờ mạng} = 10,000 \times 10\text{ ms} = 100,000\text{ ms} = \mathbf{100\text{ giây}}!$$

Các bạn thấy sự nguy hiểm chưa? Server không hề bị nghẽn CPU, đĩa cứng không hề bị quá tải, nhưng câu query vẫn mất gần 2 phút chỉ vì 10,000 lượt ping-pong gói tin TCP!

---

# III. Cài đặt / Hands-on code

Bây giờ, chúng ta sẽ bắt tay vào thực hành hai kịch bản:
1. Viết khối PL/pgSQL dùng Cursor để cập nhật hàng triệu bản ghi an toàn tuyệt đối mà không sợ tràn RAM hay khóa bảng dài ngày.
2. Thiết lập cụm kết nối Foreign Data Wrapper và đo đạc sự khác biệt kinh hoàng trước và sau khi tune `fetch_size`.

### Kịch bản 1: Xử lý theo lô (Chunking Batch) với Cursor trong PL/pgSQL

Giả sử chúng ta cần duyệt qua 500,000 giao dịch pending để cộng điểm thưởng và cập nhật trạng thái. Thay vì chạy một lệnh `UPDATE` khổng lồ gây lock toàn bộ bảng trong nhiều phút, ta chia nhỏ thành từng batch 5,000 dòng bằng Cursor:

```sql
-- Tạo bảng giả lập các giao dịch
DROP TABLE IF EXISTS transactions CASCADE;
CREATE TABLE transactions (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    amount NUMERIC(12,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    processed_at TIMESTAMPTZ
);

-- Sinh 500,000 giao dịch
INSERT INTO transactions (user_id, amount)
SELECT (random()*10000)::INT, (random()*500 + 10)::NUMERIC(12,2)
FROM generate_series(1, 500000);

ANALYZE transactions;
```

Khối mã PL/pgSQL duyệt và xử lý an toàn:
```sql
DO $$
DECLARE
    -- Khai báo con trỏ duyệt các dòng cần xử lý kèm khóa dòng FOR UPDATE
    cur_tx CURSOR FOR 
        SELECT id, amount 
        FROM transactions 
        WHERE status = 'PENDING' 
        FOR UPDATE;
        
    v_id INT;
    v_amount NUMERIC(12,2);
    v_counter INT := 0;
    v_batch_size CONSTANT INT := 5000;
BEGIN
    OPEN cur_tx;
    
    LOOP
        FETCH NEXT FROM cur_tx INTO v_id, v_amount;
        EXIT WHEN NOT FOUND;
        
        -- Cập nhật trực tiếp lên chính dòng con trỏ đang trỏ tới
        UPDATE transactions 
        SET status = 'PROCESSED', processed_at = CLOCK_TIMESTAMP()
        WHERE CURRENT OF cur_tx;
        
        v_counter := v_counter + 1;
        
        -- Ghi log tiến độ mỗi 50,000 dòng
        IF v_counter % 50000 = 0 THEN
            RAISE NOTICE 'Đã xử lý an toàn % giao dịch...', v_counter;
        END IF;
    END LOOP;
    
    CLOSE cur_tx;
    RAISE NOTICE 'Hoàn tất toàn bộ % giao dịch!', v_counter;
END $$;
```

---

### Kịch bản 2: Thiết lập Foreign Data Wrapper và Benchmark `fetch_size`

Chúng ta thiết lập một Foreign Data Wrapper trỏ sang một database remote (ở đây ta có thể giả lập ngay trên cùng cluster với một database khác có tên `remote_dw`):

```sql
-- Tạo database remote đóng vai trò Data Warehouse
CREATE DATABASE remote_dw;

-- Kết nối vào remote_dw để tạo bảng và sinh 500,000 dòng dữ liệu
\c remote_dw
CREATE TABLE remote_audit_logs (
    log_id BIGSERIAL PRIMARY KEY,
    service_name VARCHAR(50) NOT NULL,
    payload TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO remote_audit_logs (service_name, payload, created_at)
SELECT 
    (ARRAY['auth-svc', 'payment-svc', 'order-svc', 'noti-svc'])[floor(random()*4)+1],
    md5(random()::text) || md5(g::text),
    NOW() - (g || ' seconds')::INTERVAL
FROM generate_series(1, 500000) AS g;

ANALYZE remote_audit_logs;
```

Bây giờ quay trở lại database chính (Local) để thiết lập `postgres_fdw`:

```sql
\c production_db
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- 1. Tạo kết nối tới Foreign Server với cấu hình MẶC ĐỊNH (fetch_size = 100)
CREATE SERVER remote_dw_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (
    host '127.0.0.1', 
    port '5432', 
    dbname 'remote_dw',
    fetch_size '100'  -- Cấu hình mặc định ngầm của postgres_fdw
);

-- Tạo User Mapping
CREATE USER MAPPING FOR CURRENT_USER
SERVER remote_dw_server
OPTIONS (user 'postgres', password 'postgres');

-- Import schema từ server từ xa
IMPORT FOREIGN SCHEMA public LIMIT TO (remote_audit_logs)
FROM SERVER remote_dw_server INTO public;
```

#### Đo lường lần 1: Chạy với `fetch_size = 100`

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING ON)
SELECT service_name, COUNT(*), MAX(created_at)
FROM remote_audit_logs
GROUP BY service_name;
```

**Kết quả thực tế với `fetch_size = 100`:**
```text
HashAggregate  (cost=18540.00..18540.04 rows=4 width=48) (actual time=19842.150..19842.152 rows=4 loops=1)
  Group Key: service_name
  Buffers: shared hit=42
  ->  Foreign Scan on remote_audit_logs  (cost=100.00..16040.00 rows=500000 width=24) (actual time=2.150..19612.430 rows=500000 loops=1)
        Remote SQL: SELECT service_name, created_at FROM public.remote_audit_logs
Planning Time: 0.842 ms
Execution Time: 19842.610 ms  (~ 19.84 giây!)
```

Câu query mất tới **19.84 giây**! Database client và local server phải xử lý **5,000 vòng lặp fetch** qua network socket.

---

#### Đo lường lần 2: Tăng `fetch_size` lên 10,000

Bây giờ, chúng ta thay đổi cấu hình `fetch_size` của Foreign Server lên **10,000**:

```sql
-- Thay đổi fetch_size lên 10,000 cho toàn bộ Foreign Server
ALTER SERVER remote_dw_server OPTIONS (SET fetch_size '10000');

-- Hoặc có thể áp dụng riêng cho từng bảng ngoại lai nếu muốn:
-- ALTER FOREIGN TABLE remote_audit_logs OPTIONS (SET fetch_size '10000');
```

Chạy lại chính xác câu query kiểm tra:
```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING ON)
SELECT service_name, COUNT(*), MAX(created_at)
FROM remote_audit_logs
GROUP BY service_name;
```

**Kết quả thực tế sau khi tune `fetch_size = 10000`:**
```text
HashAggregate  (cost=18540.00..18540.04 rows=4 width=48) (actual time=1214.320..1214.322 rows=4 loops=1)
  Group Key: service_name
  Buffers: shared hit=42
  ->  Foreign Scan on remote_audit_logs  (cost=100.00..16040.00 rows=500000 width=24) (actual time=0.915..1085.120 rows=500000 loops=1)
        Remote SQL: SELECT service_name, created_at FROM public.remote_audit_logs
Planning Time: 0.612 ms
Execution Time: 1214.780 ms  (~ 1.21 giây!)
```

Thời gian thực thi giảm thẳng đứng từ **19.84 giây xuống còn 1.21 giây** — tốc độ **tăng hơn 16 lần** mà không cần thay đổi một dòng code ứng dụng nào!

---

#### Tối ưu nâng cao: Kích hoạt Aggregate Pushdown

Thậm chí các bạn có thể tối ưu hơn nữa. Tại sao chúng ta lại phải kéo cả 500,000 dòng về Local rồi mới làm phép `GROUP BY service_name`? Sao không bắt Remote Server tự tính toán rồi chỉ trả về đúng **4 dòng kết quả cuối cùng**?

Hãy bật cờ `pushdown` trên Foreign Server:
```sql
ALTER SERVER remote_dw_server OPTIONS (ADD use_remote_estimate 'true');

-- Đảm bảo extension cho phép pushdown aggregate
-- (Mặc định postgres_fdw trên PG 14+ tự động pushdown các hàm aggregate cơ bản như COUNT, SUM, MAX, MIN)
EXPLAIN (ANALYZE, BUFFERS)
SELECT service_name, COUNT(*), MAX(created_at)
FROM remote_audit_logs
GROUP BY service_name;
```

**Kết quả sau khi kích hoạt Pushdown:**
```text
Foreign Scan  (cost=102.50..145.20 rows=4 width=48) (actual time=85.120..85.122 rows=4 loops=1)
  Output: service_name, (count(*)), (max(created_at))
  Relations: Aggregate on (public.remote_audit_logs)
  Remote SQL: SELECT service_name, count(*), max(created_at) FROM public.remote_audit_logs GROUP BY 1
Planning Time: 2.140 ms
Execution Time: 85.340 ms  (~ 0.085 giây!)
```
Từ 19.8 giây ban đầu, qua 2 bước điều chỉnh:
1. Tune `fetch_size` Cursor: Giảm xuống 1.2 giây.
2. Kích hoạt Pushdown: Giảm xuống còn **85 mili-giây** (nhanh hơn **230 lần**)!

---

# IV. Lesson learned / Tổng kết

Cursor và Foreign Data Wrapper là hai mảnh ghép công nghệ cực kỳ mạnh mẽ nếu các bạn hiểu rõ bản chất vật lý của chúng:

1. **Cursor sinh ra để bảo vệ RAM, không phải để tăng tốc xử lý:**  
   Đừng cố gắng thay thế các truy vấn `JOIN`, `UPDATE` tập hợp bằng Cursor nếu bảng dữ liệu của bạn nằm gọn trong bộ nhớ. Hãy chỉ sử dụng Cursor khi viết các background job cần duyệt qua hàng chục triệu bản ghi hoặc stream dữ liệu ra file CSV/S3 để tránh tình trạng tràn bộ nhớ RAM (OOM).

2. **Luôn giải phóng Cursor trong khối EXCEPTION:**  
   Nếu các bạn dùng `DECLARE` và `OPEN` thủ công trong các transaction kéo dài, một lỗi runtime không được catch có thể khiến Cursor và các snapshot buffer bị treo vĩnh viễn (Portal Leak), ngăn cản tiến trình `VACUUM` dọn rác và gây phình to database (Table Bloat). Luôn đảm bảo `CLOSE cursor;` được gọi trong mọi kịch bản thoát.

3. **Kiểm tra ngay `fetch_size` trên toàn bộ Foreign Server:**  
   Con số mặc định `100` của `postgres_fdw` là một di sản an toàn thời cổ xưa để tránh tốn RAM. Trong thời đại hạ tầng đám mây mạng Gigabit ngày nay, hãy mạnh dạn đặt `fetch_size` từ **5,000 đến 10,000** ở cấp độ `FOREIGN SERVER`. Con số này mang lại sự cân bằng hoàn hảo giữa thông lượng mạng và lượng RAM chiếm dụng trên Local server.

4. **Tận dụng tối đa Query Pushdown:**  
   Luôn dùng `EXPLAIN` để kiểm tra dòng `Remote SQL` của Foreign Scan. Nếu thấy PostgreSQL đang kéo cả bảng về để tự lọc `WHERE` hoặc tự `GROUP BY` ở local, hãy kiểm tra lại kiểu dữ liệu của các hàm (chỉ những hàm `IMMUTABLE` mới được remote pushdown) và bật tùy chọn `use_remote_estimate = true` để remote database làm toàn bộ việc nặng trước khi gửi kết quả về!
