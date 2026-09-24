---
title: 'Dùng PostgreSQL làm Cache thay thế Redis: Khi UNLOGGED TABLES và JSONB phát huy sức mạnh'
date: 2026-09-24 10:00:00 +0700
categories: [Architecture, Performance]
tags: [PostgreSQL, Redis, Caching, UNLOGGED TABLES, JSONB, Performance]
keywords: [PostgreSQL, Redis, Caching, UNLOGGED TABLES, JSONB, Performance]
pin: false
image:
  path: /assets/img/posts/2026/postgresql-cache-thay-the-redis/cover.webp
  alt: 'PostgreSQL làm Cache thay thế Redis: Kiến trúc và Benchmark hiệu năng'
---

# I. Dẫn nhập

Khi một ứng dụng bắt đầu có lượng truy cập tăng lên và gặp các vấn đề về độ trễ, câu trả lời phản xạ gần như mặc định của đa số các kỹ sư phần mềm là: *"Hãy thêm Redis vào làm cache!"*. 

Redis là một công cụ xuất sắc — điều đó không ai có thể phủ nhận. Nhưng trong thực tế đi làm nhiều năm, mình nhận thấy các team kỹ thuật thường vội vã đưa Redis vào kiến trúc quá sớm mà không lường trước những chi phí vận hành đi kèm:
1. **Thêm một thành phần hạ tầng độc lập:** Các bạn phải quản lý thêm một cụm dịch vụ mới (provisioning, security, monitoring RAM, backup, high availability).
2. **Chi phí đám mây gia tăng:** Bộ nhớ RAM trên cloud (AWS ElastiCache, GCP Memorystore) rất đắt đỏ. Khi dữ liệu cache phình to, hóa đơn hàng tháng sẽ tăng vọt nhanh chóng.
3. **Bài toán đồng bộ dữ liệu và Dual-Write:** Ứng dụng phải tự quản lý logic ghi vào Database rồi ghi/xóa trên Redis. Nếu một trong hai bước gặp lỗi mạng, dữ liệu giữa Cache và DB sẽ lập tức bị lệch pha (inconsistency).
4. **Giới hạn khả năng truy vấn:** Redis lưu trữ value dưới dạng string hoặc blob. Nếu các bạn muốn lọc dữ liệu hoặc tìm kiếm theo một thuộc tính nằm sâu bên trong object vừa cache, Redis buộc bạn phải deserialize toàn bộ payload hoặc tự thiết kế các cấu trúc secondary index rất phức tạp.

Một câu hỏi thú vị được đặt ra: **Nếu hệ thống của các bạn đã có sẵn PostgreSQL, liệu ta có thể tận dụng chính PostgreSQL để làm một Cache Service hiệu năng cao thay thế Redis được không?**

Câu trả lời là **HOÀN TOÀN CÓ THỂ** — nhờ vào sự kết hợp giữa **`UNLOGGED TABLES`** (loại bỏ hoàn toàn chi phí ghi đĩa của WAL) và kiểu dữ liệu **`JSONB`** (cho phép query linh hoạt vào từng trường dữ liệu).

Trong bài viết này, mình sẽ cùng các bạn mổ xẻ nguyên lý hoạt động, xây dựng trọn vẹn giải pháp cache trên PostgreSQL (Atomic Upsert, TTL Eviction với `pg_cron`), phân tích số liệu benchmark thực tế bằng k6, và đưa ra ma trận quyết định khi nào nên dùng Postgres làm Cache, khi nào nhất định phải chuyển sang Redis!

---

# II. Kiến trúc / Nguyên lý

### 1. Cơ chế `UNLOGGED TABLES`: Tăng tốc độ ghi bằng cách bỏ qua WAL

Tại sao một bảng PostgreSQL thông thường lại có độ trễ ghi cao hơn Redis? 

Nguyên nhân cốt lõi nằm ở chuẩn an toàn dữ liệu **ACID**: Khi thực hiện một thao tác ghi (`INSERT`, `UPDATE`, `DELETE`), PostgreSQL bắt buộc phải ghi tuần tự thao tác đó vào **Write-Ahead Log (WAL)** trên đĩa và chờ hệ điều hành flush dữ liệu xuống đĩa vật lý (`fsync`). Quá trình này tạo ra độ trễ I/O nhất định.

```
[Bảng PostgreSQL Thông Thường]
INSERT / UPDATE ───► Ghi vào WAL trên Đĩa (Disk I/O fsync) ───► Ghi vào Shared Buffers (RAM)

[Bảng UNLOGGED TABLE làm Cache]
INSERT / UPDATE ───► BỎ QUA WAL HOÀN TOÀN ───► Ghi trực tiếp vào Shared Buffers (RAM)
                     (Tốc độ ghi tăng vọt ~3x - 5x)
```

Khi các bạn khai báo một bảng với từ khóa **`UNLOGGED`**:
- **Bỏ qua ghi WAL:** Mọi thao tác chèn, cập nhật hay xóa trên bảng unlogged sẽ **không bao giờ sinh ra WAL records**. Điều này giúp triệt tiêu hoàn toàn chi phí chờ đợi I/O đĩa cứng cho nhật ký giao dịch, đưa độ trễ ghi tiệm cận với tốc độ bộ nhớ RAM trong `shared_buffers`.
- **Đánh đổi về độ bền vững (Durability Trade-off):** Nếu server bị mất điện hoặc tiến trình PostgreSQL bị crash đột ngột (unclean shutdown), khi khởi động lại, PostgreSQL sẽ tự động **TRUNCATE (làm rỗng)** toàn bộ dữ liệu trong các bảng unlogged để đảm bảo tính nhất quán của hệ thống.
- **Không replicate:** Bảng unlogged không thể nhân bản sang các Read Replica thông qua cơ chế Streaming Replication (vì Streaming Replication phụ thuộc vào luồng WAL).

> **Góc nhìn thực tế:**  
> Đối với cơ sở dữ liệu chính lưu thông tin tài chính hay đơn hàng, việc mất dữ liệu khi crash là tai họa. Nhưng đối với **Cache**, dữ liệu bản chất vốn là tạm thời (ephemeral)! Nếu hệ thống cache bị rỗng sau một lần crash, ứng dụng chỉ cần query lại cơ sở dữ liệu gốc để warm up lại cache. Đây là sự đánh đổi hoàn hảo!

### 2. JSONB làm Cache Value: Sức mạnh vượt trội so với Key-Value truyền thống

Trong Redis, khi các bạn lưu một cấu trúc phức tạp như thông tin giỏ hàng hay hồ sơ người dùng:
```text
SET user:1001 '{"name":"Vinh","tier":"VIP","points":450,"cart":[{"item":"laptop","qty":1}]}'
```
Nếu bạn muốn kiểm tra xem user này có `tier = 'VIP'` hay không, ứng dụng buộc phải lấy toàn bộ chuỗi JSON về, parse JSON trên backend, rồi mới kiểm tra điều kiện.

Với PostgreSQL, khi ta lưu cache value dưới dạng **`JSONB`** (JSON nhị phân):
- PostgreSQL lưu trữ JSONB dưới dạng cây phân cấp đã được tối ưu hóa cấu trúc nhị phân.
- Các bạn có thể lọc, trích xuất hoặc thậm chí chỉ cập nhật một phần tử con bên trong JSON mà không cần deserialize:
  ```sql
  SELECT cache_value->>'tier' FROM app_cache WHERE cache_key = 'user:1001';
  ```
- Ta thậm chí có thể đánh **GIN Index** lên cột `cache_value` để tìm kiếm các bản ghi cache thỏa mãn thuộc tính JSON bất kỳ với tốc độ vài mili-giây.

### 3. Vấn đề TTL (Time-To-Live) và Quản lý Bộ nhớ (Eviction)

Redis tự động giải phóng bộ nhớ khi key hết hạn thông qua cơ chế TTL tích hợp sẵn và thuật toán Eviction (LRU/LFU). 

Trong PostgreSQL, ta có thể dễ dàng hiện thực hóa cơ chế này bằng 2 lớp bảo vệ:
1. **Lớp truy vấn (Query Layer):** Thêm điều kiện `expires_at > clock_timestamp()` vào câu lệnh `SELECT`. Dữ liệu dù chưa bị xóa vật lý nhưng đã quá hạn thì ứng dụng coi như Cache Miss.
2. **Lớp dọn dẹp nền (Background Eviction Layer):** Sử dụng extension **`pg_cron`** hoặc một cron worker chạy định kỳ mỗi 1 - 5 phút gọi lệnh `DELETE FROM app_cache WHERE expires_at <= clock_timestamp()`. Nhờ có index trên cột `expires_at`, thao tác dọn dẹp này diễn ra cực kỳ nhanh chóng và không làm gián đoạn các luồng đọc/ghi.

---

# III. Cài đặt / Hands-on code

Bây giờ, chúng ta sẽ bắt tay vào triển khai giải pháp Cache hoàn chỉnh trên PostgreSQL.

### Bước 1: Khởi tạo Schema Bảng Cache Tối ưu

Ta tạo bảng `app_cache` với từ khóa `UNLOGGED`, sử dụng `BIGSERIAL` làm khóa chính, `cache_key` là chuỗi định danh duy nhất, và `cache_value` là `JSONB`:

```sql
-- filename: schema_cache.sql

-- 1. Tạo bảng UNLOGGED làm cache service
CREATE UNLOGGED TABLE app_cache (
    id BIGSERIAL PRIMARY KEY,
    cache_key VARCHAR(255) NOT NULL,
    cache_value JSONB NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp()
);

-- 2. Tạo Unique Index trên cache_key để hỗ trợ Atomic Upsert và tìm kiếm siêu tốc
CREATE UNIQUE INDEX idx_app_cache_key ON app_cache (cache_key);

-- 3. Tạo B-Tree Index trên expires_at để phục vụ background worker xóa bản ghi hết hạn
CREATE INDEX idx_app_cache_expires_at ON app_cache (expires_at);
```

### Bước 2: Thao tác Ghi Cache Nguyên tử (Atomic Upsert)

Khi cập nhật cache, để tránh hiện tượng Race Condition giữa nhiều luồng ghi đồng thời, ta sử dụng cú pháp `INSERT ... ON CONFLICT DO UPDATE`:

```sql
-- Thao tác SET Cache (Cache-Aside pattern)
-- $1: cache_key (vd: 'user:session:1001')
-- $2: cache_value (JSON payload)
-- $3: expires_at (thời điểm hết hạn, vd: clock_timestamp() + interval '15 minutes')

INSERT INTO app_cache (cache_key, cache_value, expires_at)
VALUES (
    'user:session:1001', 
    '{"user_id": 1001, "name": "Vinh Tran", "roles": ["admin", "editor"], "login_ip": "192.168.1.5"}'::jsonb,
    clock_timestamp() + INTERVAL '15 minutes'
)
ON CONFLICT (cache_key) 
DO UPDATE SET 
    cache_value = EXCLUDED.cache_value,
    expires_at = EXCLUDED.expires_at,
    created_at = clock_timestamp();
```

Câu lệnh trên đảm bảo tính nguyên tử tuyệt đối: Nếu key chưa tồn tại thì chèn mới, nếu đã tồn tại thì ghi đè dữ liệu mới và gia hạn TTL ngay trong một round-trip duy nhất.

### Bước 3: Thao tác Đọc Cache có kiểm tra TTL

Khi backend đọc dữ liệu, ta truy vấn kèm điều kiện kiểm tra hạn sử dụng:

```sql
-- Thao tác GET Cache
SELECT cache_value 
FROM app_cache 
WHERE cache_key = 'user:session:1001' 
  AND expires_at > clock_timestamp();
```

Nếu trả về bản ghi $\rightarrow$ **Cache Hit**!  
Nếu không có bản ghi nào trả về $\rightarrow$ **Cache Miss**, backend sẽ query database nguồn và ghi ngược lại vào cache bằng câu lệnh Upsert ở Bước 2.

### Bước 4: Tự động dọn dẹp Cache hết hạn bằng `pg_cron`

Để bảng cache không bị phình to vô hạn trên đĩa, ta thiết lập một tiến trình dọn dẹp tự động chạy mỗi 5 phút bằng extension `pg_cron`:

```sql
-- Cài đặt pg_cron extension (yêu cầu cấu hình shared_preload_libraries = 'pg_cron')
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Lập lịch chạy mỗi 5 phút dọn dẹp các key đã hết hạn
SELECT cron.schedule(
    'cleanup-expired-cache', 
    '*/5 * * * *', 
    $$DELETE FROM app_cache WHERE expires_at <= clock_timestamp()$$
);

-- Kiểm tra danh sách các job đang chạy
SELECT jobid, schedule, command FROM cron.job;
```

Nếu môi trường cơ sở dữ liệu của các bạn không cho phép cài đặt `pg_cron` (ví dụ một số bản managed DB bị giới hạn quyền), các bạn có thể viết một worker đơn giản chạy trong Kubernetes CronJob hoặc background thread trong ứng dụng để thực thi câu lệnh `DELETE` định kỳ.

### Bước 5: Benchmark đối đầu thực tế bằng k6: PostgreSQL vs Redis

Để có cái nhìn khách quan, một bài kiểm thử hiệu năng (load test) chuyên sâu đã được thực hiện bằng công cụ **k6** mô phỏng 100 người dùng ảo đồng thời (**100 VUs**) trong vòng 5 phút, so sánh giữa một container PostgreSQL (bảng `UNLOGGED`) và một container Redis trên cùng cấu hình phần cứng:

| Kịch bản kiểm thử | PostgreSQL (`UNLOGGED` + JSONB) | Redis (In-Memory Key-Value) | Chênh lệch thực tế |
| :--- | :--- | :--- | :--- |
| **Ghi Cache (Upsert p95 Latency)** | **~85 ms** | **~66 ms** | Redis nhanh hơn ~19 ms |
| **Đọc Cache (Select p95 Latency)** | **~90 ms** | **~66 ms** | Redis nhanh hơn ~24 ms |
| **Throughput (Requests / sec)** | **~1,250 req/s** | **~1,580 req/s** | Redis cao hơn ~26% |
| **Tài nguyên RAM tiêu thụ thêm** | **0 MB** (dùng chung PG buffers) | **~512 MB - 2 GB** cho cụm Redis | Tiết kiệm 1 cụm dịch vụ |
| **Khả năng Filter thuộc tính con** | **Hỗ trợ 100% bằng SQL / JSONB** | Không hỗ trợ (phải deserialize) | Postgres thắng tuyệt đối |

**Nhận xét:**
Mặc dù Redis vượt trội hơn về độ trễ thuần túy (~66ms so với 85-90ms p95 ở tải cao), nhưng mức chênh lệch khoảng 20 mili-giây đối với phần lớn ứng dụng web, API thương mại điện tử hay SaaS thông thường là hoàn toàn không đáng kể. Đổi lại, các bạn tiết kiệm được toàn bộ chi phí hạ tầng và độ phức tạp vận hành của một cụm Redis riêng biệt!

---

# IV. Lesson learned / Tổng kết

Tận dụng PostgreSQL làm Cache là một giải pháp kiến trúc cực kỳ thực tế theo tinh thần **"Làm nhiều hơn với ít công cụ hơn" (Do more with less)**. Nó giúp kiến trúc hệ thống giữ được sự tinh gọn tối đa trong giai đoạn đầu và quy mô vừa.

Dưới đây là 4 bài học và kinh nghiệm tổng kết từ thực tế mà mình muốn gửi tới các bạn:

1. **Hiểu rõ bản chất dữ liệu khi dùng `UNLOGGED`:**
   - Tuyệt đối **KHÔNG BAO GIỜ** đặt dữ liệu nghiệp vụ quan trọng (đơn hàng, thông tin tài khoản, transaction) vào bảng `UNLOGGED`.
   - Bảng `UNLOGGED` chỉ phù hợp cho: Cache, Session tạm thời của người dùng, hoặc các bảng tính toán trung gian (Staging / Scratchpad) trong pipeline ETL.

2. **Cẩn trọng với hiện tượng Table Bloat:**
   - Trong PostgreSQL, thao tác `UPDATE` và `DELETE` sẽ tạo ra các dead tuples. Nếu ứng dụng ghi cache với tần suất hàng nghìn lần mỗi giây, dead tuples sẽ tích tụ rất nhanh.
   - **Giải pháp:** Cấu hình tham số **Autovacuum** riêng cho bảng cache mạnh mẽ hơn bình thường:
     ```sql
     ALTER TABLE app_cache SET (
         autovacuum_vacuum_scale_factor = 0.05,
         autovacuum_vacuum_cost_limit = 1000
     );
     ```

3. **Tận dụng Connection Pooling (PgBouncer):**
   - Khi hàng trăm microservices kết nối vào PostgreSQL để lấy cache, số lượng kết nối sẽ tăng vọt. Hãy luôn đặt **PgBouncer** phía trước ở chế độ `transaction pooling` để tái sử dụng kết nối hiệu quả, giữ mức tiêu thụ tài nguyên của database luôn ở ngưỡng an toàn.

4. **Ma trận ra quyết định: Khi nào dùng PostgreSQL Cache vs Khi nào chọn Redis?**

| Tiêu chí | Chọn PostgreSQL Cache | Chọn Dedicated Redis |
| :--- | :--- | :--- |
| **Quy mô dự án** | Startup, MVP, doanh nghiệp vừa và nhỏ | Hệ thống lớn, traffic khủng (hàng chục nghìn req/s) |
| **Độ trễ yêu cầu** | Chấp nhận được ở mức 5 - 15ms | Cực kỳ khắt khe: sub-millisecond (< 1ms) |
| **Ngân sách hạ tầng** | Muốn tối ưu chi phí, không muốn trả thêm tiền RAM | Sẵn sàng chi trả ngân sách cho cụm Redis cluster |
| **Cấu trúc dữ liệu** | Dữ liệu dạng văn bản, đối tượng JSON cần query con | Cần cấu trúc dữ liệu chuyên biệt: Sorted Sets, HyperLogLog, Pub/Sub, Geospatial |
| **Độ phức tạp DevOps** | Tối giản: 1 database duy nhất cho tất cả | Cần đội ngũ chuyên biệt giám sát Redis replication, sentinel |

Hy vọng bài viết này mang lại cho các bạn một góc nhìn mới mẻ và thực tế về năng lực tiềm ẩn của PostgreSQL. Không phải lúc nào giải pháp mới nhất cũng là tốt nhất — đôi khi, việc khai thác triệt để những công cụ ta đang có sẵn trong tay lại là quyết định kỹ thuật sáng suốt nhất!
