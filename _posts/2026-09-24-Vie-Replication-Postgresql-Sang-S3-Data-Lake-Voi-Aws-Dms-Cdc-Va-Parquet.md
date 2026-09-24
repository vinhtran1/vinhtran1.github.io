---
title: 'Continuous Replication từ PostgreSQL sang S3 Data Lake với AWS DMS: Kiến trúc CDC WAL và tối ưu định dạng Parquet'
date: 2026-09-24 13:30:00 +0700
categories: [Data Engineering, Cloud Architecture]
tags: [AWS DMS, PostgreSQL, Data Lake, Change Data Capture, Apache Parquet, AWS S3]
keywords: [AWS DMS, PostgreSQL, Data Lake, Change Data Capture, Apache Parquet]
pin: false
image:
  path: /assets/img/posts/2026/replication-postgresql-sang-s3-data-lake-voi-aws-dms-cdc-va-parquet/cover.webp
  alt: 'Kiến trúc Continuous Replication từ PostgreSQL sang Amazon S3 qua AWS DMS với CDC WAL và định dạng Parquet'
---

# I. Dẫn nhập: Đặt vấn đề và Thách thức kỹ thuật

Tại rất nhiều tổ chức công nghệ, cơ sở dữ liệu giao dịch trực tuyến (OLTP) như PostgreSQL thường là trái tim vận hành lưu trữ thông tin đơn hàng, khách hàng và giao dịch tài chính. Tuy nhiên, khi nhu cầu phân tích dữ liệu, huấn luyện mô hình Machine Learning và xây dựng Data Lakehouse bùng nổ, các kỹ sư dữ liệu thường đứng trước một bài toán nan giải: **Làm thế nào để đưa toàn bộ dữ liệu từ PostgreSQL lên Amazon S3 liên tục với độ trễ thấp nhất mà không làm ảnh hưởng đến hiệu năng của ứng dụng đang phục vụ người dùng cuối?**

Trước đây, phương pháp truyền thống phổ biến nhất là chạy các tác vụ Batch ETL định kỳ (ví dụ mỗi giờ hoặc mỗi đêm) bằng các câu truy vấn trích xuất:
```sql
-- Phương pháp trích xuất Batch truyền thống dựa trên timestamp
SELECT * FROM public.orders WHERE updated_at >= NOW() - INTERVAL '1 hour';
```

Thế nhưng, cách làm này bộc lộ những lỗ hổng chết người khi quy mô hệ thống tăng trưởng:
1. **Không thể bắt được thao tác xóa cứng (`DELETE`)**: Nếu một bản ghi bị xóa khỏi PostgreSQL (`DELETE FROM orders WHERE id = 12345;`), câu lệnh quét theo `updated_at` ở trên sẽ hoàn toàn "mù tịt" về sự kiện này. Dữ liệu trên Data Lake sẽ mãi mãi tồn tại bản ghi đã bị xóa, dẫn tới sai lệch số liệu tài chính nghiêm trọng!
2. **Quá tải tài nguyên database (Resource Contention)**: Mỗi khi tác vụ trích xuất batch kích hoạt, câu truy vấn sẽ buộc PostgreSQL quét hàng triệu dòng, đọc dữ liệu từ đĩa và đẩy văng các trang dữ liệu "nóng" (hot pages) ra khỏi bộ nhớ đệm `shared_buffers`. Hậu quả là CPU của máy chủ RDS vọt lên 90% – 100%, gây chậm trễ cho các giao dịch thanh toán của khách hàng.
3. **Độ trễ cao (High Latency)**: Báo cáo kinh doanh luôn đi sau thực tế từ 1 giờ đến 24 giờ.

```
[So sánh tác động tài nguyên trên PostgreSQL OLTP]
Truy vấn Batch (SELECT ... WHERE updated_at):
Postgres CPU  : [████████████████████████████████] 94.8% Spikes! (Table Scans & Buffer Purging)
Disk I/O      : [██████████████████████          ] High Read I/O Ops

Giải mã WAL CDC (AWS DMS Reading Replication Slot):
Postgres CPU  : [██                              ] 4.2% (Zero direct table queries!)
Disk I/O      : [█                               ] Minimal sequential disk read on WAL stream
```

Giải pháp tối thượng cho vấn đề này là kỹ thuật **Change Data Capture (CDC) thời gian thực thông qua AWS Database Migration Service (AWS DMS)**. Thay vì truy vấn trực tiếp vào các bảng dữ liệu, AWS DMS sẽ đóng vai trò như một client sao chép logic (Logical Replication Consumer), đọc tuần tự từ **Write-Ahead Logging (WAL)** của PostgreSQL và liên tục đồng bộ dữ liệu sang Amazon S3 dưới định dạng cột nén **Apache Parquet**.

Trong bài viết này, mình sẽ cùng các bạn mổ xẻ nguyên lý giải mã WAL trong PostgreSQL, cơ chế quản lý Replication Slot, cấu hình chi tiết Extra Connection Attributes (ECA) để ghi file Parquet chuẩn mực, cách xử lý bài toán "thảm họa file nhỏ" (Small Files Problem), và thiết lập View hợp nhất trạng thái mới nhất trên Amazon Athena.

---

# II. Kiến trúc / Nguyên lý cốt lõi

## 1. Cơ chế Logical Replication và Giải mã WAL trong PostgreSQL

Trong kiến trúc nội tại của PostgreSQL, mọi thao tác sửa đổi dữ liệu (`INSERT`, `UPDATE`, `DELETE`) đều phải được ghi tuần tự vào nhật ký **Write-Ahead Log (WAL)** trước khi được đẩy xuống các tệp dữ liệu chính (`base/` directory). Các file WAL được chia thành từng phân đoạn (segment) cố định với kích thước 16MB.

Khi chúng ta cấu hình tham số `wal_level = logical`:
1. PostgreSQL không chỉ ghi lại các thay đổi nhị phân ở mức block đĩa, mà còn bổ sung thêm thông tin ngữ nghĩa logic ở mức hàng (Row-level information): Tên bảng, cấu trúc cột, giá trị của các cột trước và sau khi thay đổi.
2. Để theo dõi tiến trình đọc, PostgreSQL tạo ra một đối tượng gọi là **Logical Replication Slot**. Replication Slot lưu trữ một con trỏ số thứ tự nhật ký **Log Sequence Number (LSN)** đại diện cho vị trí xa nhất mà tiến trình đọc (ở đây là AWS DMS) đã xử lý và xác nhận (`confirmed_flush_lsn`).

```
[PostgreSQL Engine] ---> Ghi nhận thay đổi giao dịch vào ---> [ pg_wal Segments ]
                                                                     |
                                                                     | Đọc stream WAL
                                                                     v
                                                   +------------------------------------+
                                                   | Logical Replication Slot (AWS DMS) |
                                                   | Confirmed LSN: 0/1A3B4C5D          |
                                                   +------------------------------------+
```

### Cạm bẫy sống còn: Nguy cơ tràn đĩa WAL (WAL Disk Overflow Hazard)
Nguyên tắc bất di bất dịch của PostgreSQL là: **Không bao giờ xóa các file WAL mà Replication Slot chưa xác nhận đã đọc xong!**

Hãy hình dung kịch bản sau: AWS DMS Replication Task gặp sự cố mạng hoặc bị dừng (Stopped), nhưng các bạn không hề hay biết và để nguyên trạng thái đó trong 3 ngày. Trong khi đó, hệ thống OLTP vẫn liên tục phát sinh giao dịch:
- PostgreSQL muốn dọn dẹp các file WAL cũ nhưng nhận thấy Replication Slot của DMS vẫn đang neo ở vị trí LSN từ 3 ngày trước.
- Hàng nghìn file WAL 16MB liên tục tích tụ trong thư mục `pg_wal`.
- Chỉ sau một thời gian ngắn, dung lượng ổ cứng của RDS chạm ngưỡng 100%. PostgreSQL lập tức rơi vào trạng thái `PANIC` và **tự động sập toàn bộ máy chủ database**!

> **Quy tắc vàng**: Khi triển khai CDC với AWS DMS, các bạn bắt buộc phải thiết lập cảnh báo CloudWatch cho chỉ số `OldestReplicationSlotLag` để phát hiện ngay khi DMS bị chậm trễ hoặc ngắt kết nối.

### Tầm quan trọng của `REPLICA IDENTITY FULL`
Mặc định, khi một câu lệnh `UPDATE` hoặc `DELETE` xảy ra, PostgreSQL chỉ ghi nhận giá trị mới cùng với giá trị của Primary Key vào WAL stream. Nhưng đối với Data Lake, nếu chúng ta muốn bắt trọn vẹn toàn bộ ngữ cảnh dữ liệu cũ (Old Tuple Values) để so sánh thay đổi hoặc xử lý các bảng không có khóa chính, chúng ta cần cấu hình `REPLICA IDENTITY FULL` cho bảng đó.

## 2. Luồng dữ liệu tổng thể với AWS DMS (End-to-End Pipeline)

Kiến trúc sao chép liên tục từ PostgreSQL sang S3 thông qua AWS DMS được tổ chức thành 2 giai đoạn kế tiếp nhau:

```
+--------------------+        Logical WAL Stream       +-------------------------+
| Amazon RDS Postgres|================================>|   AWS DMS Instance      |
| (wal_level=logical)|   (Full Load + Continuous CDC)  | (dms.r5.large Multi-AZ) |
+--------------------+                                 +-------------------------+
                                                                    |
                                                                    | Snappy Parquet
                                                                    | (ECA Optimized)
                                                                    v
                                                       +-------------------------+
                                                       | Amazon S3 Raw Data Lake |
                                                       | /schema/table/YYYY/MM/..|
                                                       +-------------------------+
                                                                    |
                                                                    | Compaction / SQL
                                                                    v
                                                       +-------------------------+
                                                       |   AWS Athena / Lakehouse|
                                                       | (Deduplication Views)   |
                                                       +-------------------------+
```

### Hai giai đoạn của Replication Task:
1. **Pha 1: Full Load (Nạp toàn bộ dữ liệu ban đầu)**:
   - DMS mở kết nối SELECT thông thường tới database nguồn để trích xuất toàn bộ dữ liệu hiện có thành các file Parquet ban đầu trên S3.
   - Trong lúc Full Load đang chạy, DMS vẫn âm thầm mở Replication Slot để tích lũy các thay đổi WAL phát sinh trong quá trình quét.
2. **Pha 2: Ongoing Replication / CDC (Sao chép thay đổi liên tục)**:
   - Sau khi hoàn thành Full Load, DMS chuyển sang đọc luồng WAL stream từ Replication Slot.
   - Các bản ghi thay đổi được nạp vào bộ nhớ đệm (buffer) của DMS Instance, chuyển đổi định dạng và ghi định kỳ thành các tệp Parquet mới trên S3.

## 3. Cấu hình Extra Connection Attributes (ECA) cho S3 Target

Để dữ liệu trên S3 đạt hiệu năng cao nhất cho các công cụ truy vấn như Athena, Presto hay Spark, chúng ta không dùng định dạng CSV thô mà cấu hình S3 Target Endpoint xuất thẳng ra định dạng cột nén **Apache Parquet**.

Các thuộc tính mở rộng (Extra Connection Attributes - ECA) đóng vai trò quyết định chất lượng của tệp Parquet:
- `dataFormat=parquet;`: Chỉ định ghi dữ liệu dạng Parquet thay vì CSV.
- `parquetVersion=PARQUET_2_0;`: Sử dụng phiên bản Parquet 2.0 hiện đại với khả năng mã hóa (encoding) tối ưu.
- `compressionType=SNAPPY;`: Chuẩn nén Snappy mang lại tỷ lệ nén tốt và tốc độ giải nén cực nhanh cho query engine.
- `includeOpIndicator=true;`: Thêm một trường siêu dữ liệu có tên là `Op` vào file Parquet. Cột này sẽ chứa giá trị `'I'` (Insert), `'U'` (Update), hoặc `'D'` (Delete).
- `datePartitionedEnabled=true; datePartitionDelimiter=SLASH;`: Tự động phân chia thư mục S3 theo năm/tháng/ngày (`YYYY/MM/DD`), giúp Athena chỉ quét các phân vùng thời gian cần thiết.

---

# III. Cài đặt / Hands-on code: Hiện thực & Tối ưu thực chiến

## 1. Cấu hình PostgreSQL Database làm CDC Source

Đầu tiên, các bạn cần cập nhật DB Parameter Group của Amazon RDS PostgreSQL và khởi động lại database (nếu cần):

```ini
# Cấu hình Parameter Group cho PostgreSQL
rds.logical_replication = 1     # Kích hoạt wal_level = logical
wal_sender_timeout = 0          # Tránh ngắt kết nối replication khi DMS xử lý batch lớn
max_replication_slots = 10      # Số lượng slot tối đa cho các consumer
max_wal_senders = 10            # Số lượng tiến trình truyền WAL stream
```

Sau đó, kết nối vào PostgreSQL bằng tài khoản quản trị viên (`postgres`) và thực thi các lệnh cấp quyền chuẩn mực:

```sql
-- 1. Tạo tài khoản riêng biệt dành cho AWS DMS
CREATE USER dms_cdc_user WITH PASSWORD 'UltraSecurePassword_2026!';

-- 2. Cấp quyền truy cập schema và bảng dữ liệu
GRANT USAGE ON SCHEMA public TO dms_cdc_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO dms_cdc_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO dms_cdc_user;

-- 3. Cấp role chuyên dụng cho replication trên AWS RDS
GRANT rds_replication TO dms_cdc_user;

-- 4. Bật REPLICA IDENTITY FULL cho các bảng trọng yếu cần CDC
ALTER TABLE public.orders REPLICA IDENTITY FULL;
ALTER TABLE public.order_items REPLICA IDENTITY FULL;
ALTER TABLE public.customers REPLICA IDENTITY FULL;
```

## 2. Khởi tạo AWS DMS Endpoints bằng Terraform

Dưới đây là đoạn mã Terraform mẫu định nghĩa Source Endpoint và Target S3 Endpoint với đầy đủ bộ thuộc tính tối ưu hóa:

```hcl
# 1. Source Endpoint: Kết nối tới PostgreSQL
resource "aws_dms_endpoint" "postgres_source" {
  endpoint_id                 = "rds-postgres-production-source"
  endpoint_type               = "source"
  engine_name                 = "postgres"
  username                    = "dms_cdc_user"
  password                    = "UltraSecurePassword_2026!"
  server_name                 = "production-db.internal-vpc.local"
  port                        = 5432
  database_name               = "core_commerce"
  ssl_mode                    = "require"

  extra_connection_attributes = "captureDdls=true;pluginName=pglogical;ddlIncludes=true;"
}

# 2. Target Endpoint: Amazon S3 Data Lake với định dạng Parquet
resource "aws_dms_endpoint" "s3_parquet_target" {
  endpoint_id   = "s3-datalake-raw-parquet-target"
  endpoint_type = "target"
  engine_name   = "s3"

  s3_settings {
    service_access_role_arn = aws_iam_role.dms_s3_role.arn
    bucket_name             = "enterprise-raw-data-lake-2026"
    bucket_folder           = "cdc_stream"
    data_format             = "parquet"
    parquet_version         = "parquet-2-0"
    compression_type        = "snappy"
    
    # Kích hoạt cột cờ thao tác CDC (I, U, D)
    include_op_indicator   = true
    
    # Phân vùng ngày tháng tự động
    date_partitioned_enabled   = true
    date_partition_delimiter   = "SLASH"
    date_partition_sequence    = "YYYYMMDD"
    
    # =========================================================================
    # GIẢI PHÁP CHỐNG THẢM HỌA FILE NHỎ (SMALL FILES PROBLEM)
    # =========================================================================
    # Chỉ ghi file xuống S3 khi kích thước buffer đạt tối thiểu 32 MB
    cdc_min_file_size          = 32768
    # Hoặc ghi file nếu sau 300 giây (5 phút) mà dung lượng chưa đạt ngưỡng
    cdc_max_batch_interval     = 300
  }
}
```

## 3. Giải quyết thảm họa Small Files: Buffer Tuning

Một trong những bài học đau đớn nhất khi mới vận hành DMS sang S3 là **Thảm họa tệp nhỏ (Small Files Problem)**. Theo cơ chế mặc định, mỗi khi có vài giao dịch nạp vào, DMS sẽ lập tức đẩy một file Parquet xuống S3 với kích thước chỉ từ 10KB đến 50KB!

Chỉ sau 1 tuần, một bảng có thể sinh ra hơn **100,000 file nhỏ**:
- Chi phí lưu trữ S3 tăng vọt do số lượng request `PUT` khổng lồ.
- Khi các bạn dùng Amazon Athena hoặc Apache Spark để truy vấn, engine sẽ mất 90% thời gian chỉ để gửi request HTTP `GET` mở metadata của hàng trăm nghìn file nhỏ, khiến truy vấn bị chậm đi 20 đến 50 lần!

### Bộ đôi thông số cứu cánh trong S3 Settings:
1. `cdc_min_file_size = 32768`: Yêu cầu DMS giữ dữ liệu trong bộ nhớ RAM của Replication Instance và chỉ xuất file Parquet khi dung lượng đạt tối thiểu **32 MB** (hoặc 64 MB).
2. `cdc_max_batch_interval = 300`: Đặt ngưỡng thời gian tối đa là **300 giây (5 phút)**. Nếu sau 5 phút mà lưu lượng giao dịch ít và chưa gom đủ 32MB, DMS vẫn sẽ xả dữ liệu ra file để đảm bảo độ trễ của Data Lake không vượt quá 5 phút.

## 4. Truy vấn hợp nhất CDC trên Amazon Athena (Deduplication View)

Vì DMS ghi dữ liệu CDC dạng nối tiếp (Append-only stream), trên S3 sẽ chứa cả lịch sử Insert, Update và Delete của cùng một bản ghi. Để các nhà phân tích dữ liệu (Data Analysts) có thể truy vấn trạng thái thực tế mới nhất, chúng ta tạo một bảng ngoại vi (External Table) và một **Deduplication SQL View** trên Athena:

```sql
-- 1. Tạo External Table trỏ tới thư mục Parquet trên S3
CREATE EXTERNAL TABLE IF NOT EXISTS datalake_raw.cdc_orders (
    id BIGINT,
    customer_id BIGINT,
    order_status VARCHAR(50),
    total_amount DECIMAL(18, 2),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    Op VARCHAR(2) -- Cột chỉ số do DMS tự động chèn: 'I', 'U', 'D'
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
STORED AS INPUTFORMAT 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
LOCATION 's3://enterprise-raw-data-lake-2026/cdc_stream/public/orders/';

-- 2. Tạo View hợp nhất bản ghi mới nhất (Current State Deduplication View)
CREATE OR REPLACE VIEW datalake_analytics.view_current_orders AS
WITH ranked_orders AS (
    SELECT 
        id,
        customer_id,
        order_status,
        total_amount,
        created_at,
        updated_at,
        Op,
        ROW_NUMBER() OVER (
            PARTITION BY id 
            ORDER BY updated_at DESC, created_at DESC
        ) AS row_num
    FROM datalake_raw.cdc_orders
)
SELECT 
    id,
    customer_id,
    order_status,
    total_amount,
    created_at,
    updated_at
FROM ranked_orders
WHERE row_num = 1
  AND Op != 'D'; -- Loại bỏ hoàn toàn các bản ghi đã bị DELETE ở nguồn!
```

Giờ đây, bất kỳ câu truy vấn nào gọi `SELECT * FROM datalake_analytics.view_current_orders` đều sẽ trả về dữ liệu chuẩn xác 100% như trên PostgreSQL, tự động loại bỏ các bản ghi đã bị xóa và luôn hiển thị trạng thái mới nhất của đơn hàng!

---

# IV. Lesson learned: Tổng kết & Best Practices

Triển khai thành công kiến trúc Continuous Replication từ PostgreSQL sang S3 đòi hỏi sự thấu hiểu sâu sắc cả về cơ sở dữ liệu lẫn kiến trúc đám mây. Dưới đây là 5 bài học thực chiến các bạn cần khắc cốt ghi tâm:

1. **Giám sát Replication Slot Lag bằng PagerDuty**:
   - Đây là rủi ro hạ tầng số 1. Hãy cấu hình CloudWatch Alarm cho metric `OldestReplicationSlotLag` trên RDS PostgreSQL. Nếu độ trễ vượt quá 10 GB (hoặc 1 giờ không có tiến trình đọc), lập tức bắn cảnh báo khẩn cấp để đội ngũ kiểm tra trạng thái của DMS Replication Task trước khi đĩa cứng bị tràn.

2. **Luôn kích hoạt `REPLICA IDENTITY FULL` cho các bảng cần CDC**:
   - Không có thuộc tính này, các bản ghi `UPDATE` và `DELETE` sẽ thiếu dữ liệu ngữ cảnh của các trường không phải khóa chính, gây khó khăn cho việc tái hiện lịch sử hoặc tổng hợp dữ liệu trên Data Lake.

3. **Phân tách rõ ràng kiến trúc Bronze và Silver Data Lake**:
   - Thư mục S3 mà DMS ghi vào chỉ nên được coi là tầng Bronze (Raw Layer). 
   - Định kỳ hàng ngày hoặc hàng giờ, các bạn nên sử dụng Apache Iceberg hoặc AWS Glue / DuckDB để nén (Compaction) và sáp nhập (Merge-On-Read/Copy-On-Write) dữ liệu từ Bronze sang Silver Layer với các khối Parquet tiêu chuẩn 128MB – 256MB.

4. **Lựa chọn dòng máy tối ưu RAM cho DMS Replication Instance**:
   - Trong quá trình bắt CDC, DMS phải lưu giữ transaction buffer trong bộ nhớ RAM trước khi ghi xuống S3. Đối với các hệ thống có nhiều giao dịch lớn, tuyệt đối không dùng dòng máy nhỏ (như `t3.micro`). Hãy bắt đầu tối thiểu từ dòng `dms.r5.large` hoặc `dms.r5.xlarge` để đảm bảo DMS có đủ RAM đệm.

5. **Chủ động xử lý Schema Evolution**:
   - Khi có thêm cột mới trên bảng PostgreSQL nguồn, hãy đảm bảo tham số `captureDdls=true` đã được bật trên DMS Endpoint. Kiểm tra xem các file Parquet mới sinh ra có nhận diện đúng cột mới hay không và chạy `MSCK REPAIR TABLE` hoặc cập nhật AWS Glue Data Catalog tương ứng.
