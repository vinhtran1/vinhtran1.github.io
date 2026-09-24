---
title: 'Apache Iceberg Copy-On-Write vs Merge-On-Read: Bản chất kỹ thuật, Trade-offs và Ứng dụng thực tế trên AWS Athena'
date: 2026-09-24 11:00:00 +0700
categories: [Data Engineering, Lakehouse]
tags: [Apache Iceberg, Lakehouse, AWS Athena, Parquet, Data Engineering, Performance Tuning]
keywords: [Apache Iceberg, Lakehouse, AWS Athena, Performance Tuning]
pin: false
image:
  path: /assets/img/posts/2026/apache-iceberg-cow-vs-mor-tradeoffs-va-ung-dung-athena/cover.webp
  alt: 'So sánh Copy-On-Write và Merge-On-Read trong Apache Iceberg và tối ưu truy vấn trên AWS Athena'
---

# I. Dẫn nhập

Chào các bạn, trong kỷ nguyên của Modern Data Lakehouse, các định dạng bảng mở (Open Table Formats) như **Apache Iceberg**, **Delta Lake** và **Apache Hudi** đã tạo nên một cuộc cách mạng thực sự. Chúng giải quyết triệt để những điểm yếu chí mạng của kiến trúc Apache Hive truyền thống: mang lại tính nhất quán giao dịch ACID, cho phép Time Travel quay ngược thời gian, và hỗ trợ tiến hóa phân vùng (Partition Evolution) mượt mà ngay trên tầng lưu trữ đám mây giá rẻ như Amazon S3.

Tuy nhiên, khi đưa Apache Iceberg vào vận hành thực tế trong các hệ thống dữ liệu doanh nghiệp, thử thách hóc búa nhất mà bất kỳ Data Engineer nào cũng phải đối mặt chính là: **Xử lý các thao tác cập nhật và xóa bản ghi (`UPDATE` / `DELETE`)**. 

Thực tế đi làm không bao giờ màu hồng như trong sách giáo khoa (nơi dữ liệu chỉ có Append-only). Chúng ta liên tục phải đối mặt với:
- Dòng sự kiện Change Data Capture (CDC) từ Debezium / Kafka Connect phản ánh trạng thái đơn hàng (từ `PENDING` sang `SHIPPED` rồi `DELIVERED`).
- Yêu cầu tuân thủ pháp lý quyền riêng tư dữ liệu (GDPR / CCPA) với quy định "Right to be forgotten" — buộc phải xóa sạch thông tin người dùng khỏi Data Lake trong vòng 30 ngày.
- Pipeline xử lý Late-Arriving Data và Deduplication (khử trùng lặp) hàng triệu bản ghi mỗi ngày.

Để xử lý các biến động này, Apache Iceberg cung cấp cho chúng ta 2 triết lý đối đầu trực diện: **Copy-On-Write (COW)** và **Merge-On-Read (MOR)**.

Chọn sai chế độ này sẽ dẫn đến những thảm họa vận hành cực kỳ tốn kém:
- Nếu chọn COW cho một streaming pipeline có tần suất cập nhật cao: Write pipeline của bạn sẽ bị nghẽn (bottleneck) vì **Write Amplification** khủng khiếp. Sửa một trường giá trị 4 byte trong 1 dòng dữ liệu có thể ép engine phải đọc và ghi lại toàn bộ file Parquet dung lượng 500 MB lên S3!
- Nếu vội vã chuyển sang MOR mà không có chiến lược bảo trì (Maintenance): Dashboard báo cáo trên AWS Athena hoặc Trino sẽ chạy chậm như rùa bò vì **Read Amplification**. Mỗi câu lệnh `SELECT` của người dùng phải tốn tài nguyên CPU gom và đối soát hàng trăm Delete Files tại thời điểm truy vấn.

Trong bài viết này, mình sẽ cùng các bạn bóc tách sâu tận chân tơ kẽ tóc: cấu trúc vật lý của Data Files vs Delete Files trong Iceberg V2 Spec, phân tích trade-offs hiệu năng, và hướng dẫn cấu hình chi tiết bảng Iceberg trên AWS Athena v3 cùng kịch bản Compaction tự động bằng PySpark.

---

# II. Kiến trúc / Nguyên lý

Để hiểu được bản chất của COW và MOR, trước hết chúng ta cần nắm vững cấu trúc cây Metadata 3 tầng của Apache Iceberg: **Iceberg Catalog -> Metadata File -> Manifest List -> Manifest Files -> Data Files / Delete Files**.

```
+---------------------------------------------------------------------------------------------------+
|                                  APACHE ICEBERG V2 SPEC TOPOLOGY                                  |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|                            +------------------------------------+                                 |
|                            |        Iceberg Catalog (Glue)      |                                 |
|                            +-----------------+------------------+                                 |
|                                              |                                                    |
|                                              v                                                    |
|                            +------------------------------------+                                 |
|                            |     v2.metadata.json (Snapshot)    |                                 |
|                            +-----------------+------------------+                                 |
|                                              |                                                    |
|                                              v                                                    |
|                            +------------------------------------+                                 |
|                            |       snap-xxx.avro (Manifest List)|                                 |
|                            +--------+------------------+--------+                                 |
|                                     |                  |                                          |
|                                     v                  v                                          |
|                   +-----------------------+      +-----------------------+                        |
|                   | manifest-data.avro    |      | manifest-delete.avro  |                        |
|                   +-----------+-----------+      +-----------+-----------+                        |
|                               |                              |                                    |
|         +---------------------+--------------------+         |                                    |
|         |                                          |         |                                    |
|         v                                          v         v                                    |
|  +--------------+                           +--------------+ +--------------------+               |
|  | data_1.parquet|                           | data_2.parquet| | del_pos_1.parquet  |               |
|  | (100k rows)  |                           | (100k rows)  | | (file_path, pos)   |               |
|  +--------------+                           +--------------+ +--------------------+               |
|                                                     ^                  |                          |
|                                                     +---(MOR Anti-Join)+                          |
+---------------------------------------------------------------------------------------------------+
```

### 1. Cơ chế Copy-On-Write (COW): Tối ưu cho người đọc (Read-Optimized)
Triết lý của Copy-On-Write rất đơn giản: **Dữ liệu trên đĩa luôn ở trạng thái sẵn sàng truy vấn nhanh nhất**.

Khi có một câu lệnh `UPDATE` hoặc `DELETE` xảy ra:
1. Engine thực thi (Spark / Athena) xác định các file Parquet chứa các dòng dữ liệu bị ảnh hưởng.
2. Engine tải toàn bộ các file Parquet này vào bộ nhớ RAM.
3. Áp dụng các thay đổi: loại bỏ các dòng bị xóa, ghi đè các dòng được cập nhật.
4. Ghi ra một tập hợp các **Data Files Parquet hoàn toàn mới** lên S3.
5. Tạo một Snapshot mới trong Iceberg Metadata trỏ tới các data files mới và đánh dấu các data files cũ là không còn hoạt động.

```
COW Execution Flow:
[Data File A (v1) on S3] ──> Read into RAM ──> Apply Modifications ──> Write [Data File A' (v2) on S3]
                                                                        └─> Metadata points to A'
```

- **Ưu điểm vượt trội:**
  - **Zero Read Overhead:** Query Engine (Athena, Presto, StarRocks) chỉ cần scan trực tiếp các file Parquet hoàn chỉnh, tận dụng 100% sức mạnh của Columnar Pruning và Vectorized Execution. Không cần join thêm bất kỳ file nào.
- **Nhược điểm chí mạng:**
  - **Write Amplification cực lớn:** Nếu bảng của bạn chứa các file Parquet kích thước 512 MB và mỗi batch bạn chỉ cập nhật 10 dòng rải rác trên 100 files, engine sẽ phải đọc và ghi lại tới 50 GB dữ liệu lên S3! Điều này gây lãng phí băng thông mạng, tốn chi phí S3 PUT requests và làm chậm tốc độ của pipeline nạp dữ liệu.

### 2. Cơ chế Merge-On-Read (MOR): Tối ưu cho người ghi (Write-Optimized)
Merge-On-Read sinh ra để khắc phục nhược điểm Write Amplification của COW. Triết lý của nó là: **Khi cập nhật hoặc xóa dữ liệu, tuyệt đối không chạm vào data file cũ đã ghi**.

Thay vì viết lại toàn bộ file dữ liệu, Iceberg chỉ tạo ra một file nhẹ hơn rất nhiều gọi là **Delete File**. 

Trong đặc tả Iceberg Format V2, có 2 loại Delete Files:
1. **Position Delete Files (`content = 1`):** Lưu chính xác đường dẫn URI của file dữ liệu và số thứ tự chỉ mục dòng (row ordinal / position offset) của bản ghi bị xóa trong file đó:
   - Cột 1: `file_path = 's3://my-bucket/data/data_2.parquet'`
   - Cột 2: `pos = 4281`
2. **Equality Delete Files (`content = 2`):** Lưu trực tiếp giá trị của cột điều kiện xóa (ví dụ: `customer_id = 9999`). Loại này thường được sinh ra bởi các engine streaming như Apache Flink khi engine không biết bản ghi nằm ở file nào.

```
MOR Execution Flow:
[Data File B (v1) on S3] ──(Unchanged)──┐
                                        ├─> Athena Engine Merge/Anti-Join at Query Time ──> Result Rows
[Delete File del_1 on S3] ──────────────┘
```

- **Ưu điểm vượt trội:**
  - **Tốc độ ghi siêu tốc (Low Write Latency):** Thao tác `DELETE` chỉ tốn vài chục mili-giây để ghi một Delete File vài Kilobytes lên S3. Cực kỳ lý tưởng cho các đường ống nạp dữ liệu Streaming hoặc CDC thời gian thực.
- **Nhược điểm:**
  - **Read Amplification:** Khi người dùng truy vấn trên AWS Athena, Athena phải đọc Data File, đồng thời đọc Delete File vào RAM, thực hiện một phép nối loại trừ (Anti-Join / Dynamic Bitset Filtering) để lọc bỏ các dòng đã xóa trước khi trả kết quả cho người dùng. Nếu bảng tích lũy hàng nghìn Delete Files, query time sẽ tăng vọt theo cấp số nhân!

### 3. Ma trận Trade-offs: So sánh trực diện COW vs MOR

| Tiêu chí kỹ thuật | Copy-On-Write (COW) | Merge-On-Read (MOR) |
| :--- | :--- | :--- |
| **Write Latency** | Chậm (Phải nạp & ghi lại full file Parquet) | Cực nhanh (Chỉ ghi delta delete file nhỏ) |
| **Write Amplification** | Rất cao ($10\times - 1000\times$) | Rất thấp ($\approx 1\times$) |
| **Read Latency** | Tối ưu tuyệt đối (Direct columnar scan) | Chậm hơn (Phải merge & filter on-the-fly) |
| **Storage Cost (Daily)**| Tốn chi phí lưu trữ các file v1, v2 | Tốn ít dung lượng cho các file delete |
| **Tần suất Compaction** | Thấp (Chỉ cần gom small files) | Bắt buộc phải chạy thường xuyên |
| **Công cụ tương thích** | Hầu hết mọi engine đều đọc nhanh | Cần engine hỗ trợ Iceberg V2 (Athena v3) |

### 4. Tầm quan trọng của Table Maintenance (Compaction)
Dùng MOR mà không chạy Compaction định kỳ thì chẳng khác nào "vay nợ kỹ thuật với lãi suất cắt cổ". Càng nhiều Delete Files tích tụ, chi phí truy vấn Athena càng đắt đỏ.

Apache Iceberg cung cấp thủ tục lưu trữ (Stored Procedure) `rewrite_data_files` giúp hòa giải mâu thuẫn này:
- Định kỳ (hàng đêm hoặc mỗi 2-4 giờ), một tiến trình Spark / Glue Job sẽ quét qua bảng Iceberg, đọc các Data Files cùng các Delete Files tương ứng, thực hiện gộp (merge) triệt để và ghi ra các Data Files mới sạch sẽ.
- Toàn bộ Delete Files cũ được gỡ bỏ. Bảng Iceberg quay trở lại trạng thái đọc tối ưu 100% giống như COW!

---

# III. Cài đặt / Hands-on code

Bây giờ, mình sẽ cùng các bạn thực hành thiết lập bảng Iceberg trên **AWS Athena Engine v3** (dựa trên Trino), quan sát sự khác biệt cấu trúc vật lý của COW và MOR, và xây dựng kịch bản bảo trì Compaction bằng PySpark.

### 1. DDL Tạo Bảng Iceberg trên AWS Athena v3

Trên AWS Athena Console, đảm bảo Workgroup của bạn đang chạy **Athena engine version 3**.

```sql
-- Tạo Database thử nghiệm
CREATE DATABASE IF NOT EXISTS iceberg_demo_db;

-- ====================================================================
-- 1. BẢNG COPY-ON-WRITE (COW)
-- ====================================================================
CREATE TABLE iceberg_demo_db.orders_cow (
    order_id         BIGINT,
    customer_id      BIGINT,
    order_amount     DECIMAL(12, 2),
    order_status     VARCHAR,
    created_at       TIMESTAMP
)
LOCATION 's3://my-lakehouse-bucket/iceberg_demo/orders_cow/'
TBLPROPERTIES (
    'table_type' = 'ICEBERG',
    'format' = 'parquet',
    'write.delete.mode' = 'copy-on-write',
    'write.update.mode' = 'copy-on-write',
    'write.merge.mode'  = 'copy-on-write'
);

-- ====================================================================
-- 2. BẢNG MERGE-ON-READ (MOR)
-- ====================================================================
CREATE TABLE iceberg_demo_db.orders_mor (
    order_id         BIGINT,
    customer_id      BIGINT,
    order_amount     DECIMAL(12, 2),
    order_status     VARCHAR,
    created_at       TIMESTAMP
)
LOCATION 's3://my-lakehouse-bucket/iceberg_demo/orders_mor/'
TBLPROPERTIES (
    'table_type' = 'ICEBERG',
    'format' = 'parquet',
    'format-version' = '2',
    'write.delete.mode' = 'merge-on-read',
    'write.update.mode' = 'merge-on-read',
    'write.merge.mode'  = 'merge-on-read'
);
```

### 2. Thực hiện Thao tác Ghi và Xóa Dữ liệu

Nạp dữ liệu mẫu vào cả hai bảng:

```sql
-- Chèn 100,000 dòng dữ liệu vào cả 2 bảng
INSERT INTO iceberg_demo_db.orders_cow
SELECT 
    CAST(n AS BIGINT) AS order_id,
    CAST((n % 1000) AS BIGINT) AS customer_id,
    CAST(ROUND(random() * 500 + 10, 2) AS DECIMAL(12, 2)) AS order_amount,
    'COMPLETED' AS order_status,
    CURRENT_TIMESTAMP - interval '1' day * CAST(random() * 30 AS INT) AS created_at
FROM UNNEST(sequence(1, 100000)) AS t(n);

INSERT INTO iceberg_demo_db.orders_mor
SELECT * FROM iceberg_demo_db.orders_cow;
```

Bây giờ, hãy thực hiện câu lệnh xóa 5,000 đơn hàng:

```sql
-- Xóa các đơn hàng có order_id lẻ dưới 10,000 trên cả 2 bảng
DELETE FROM iceberg_demo_db.orders_cow WHERE order_id <= 10000 AND order_id % 2 = 1;

DELETE FROM iceberg_demo_db.orders_mor WHERE order_id <= 10000 AND order_id % 2 = 1;
```

### 3. "Soi" cấu trúc tầng ngầm qua Iceberg Metadata Tables

Điểm tuyệt vời của Apache Iceberg là cho phép ta truy vấn trực tiếp cấu trúc metadata nội bộ thông qua các bảng ảo (Metadata System Tables) với ký tự `$`.

Hãy kiểm tra bảng `$files` của bảng MOR trên Athena:

```sql
SELECT 
    content, -- 0: Data File, 1: Position Deletes, 2: Equality Deletes
    file_path,
    file_format,
    record_count,
    file_size_in_bytes
FROM "iceberg_demo_db"."orders_mor$files"
ORDER BY content DESC;
```

**Kết quả quan sát thực tế:**
- Với bảng `orders_cow`: Số lượng file dữ liệu cũ bị thay thế bởi các file Parquet mới. Cột `content` chỉ toàn giá trị `0` (Data files).
- Với bảng `orders_mor`: Xuất hiện thêm các file mới với `content = 1` (Position Delete files), dung lượng chỉ khoảng 15 KB, trỏ tới vị trí các dòng bị xóa trong file Parquet gốc.

Hãy kiểm tra bảng `$snapshots` để xem lịch sử commit:

```sql
SELECT 
    snapshot_id,
    parent_id,
    operation, -- 'append', 'delete', 'overwrite'
    summary['added-data-files'] AS added_files,
    summary['deleted-data-files'] AS deleted_files,
    summary['added-delete-files'] AS added_deletes,
    summary['total-delete-files'] AS total_deletes
FROM "iceberg_demo_db"."orders_mor$snapshots"
ORDER BY committed_at DESC;
```

### 4. Kịch bản Bảo trì Tự động (Compaction & Clean-up) với PySpark

Vì AWS Athena hiện tại chưa hỗ trợ câu lệnh gọi stored procedure `CALL system.rewrite_data_files()`, chúng ta sử dụng một PySpark Job (chạy trên AWS Glue hoặc Amazon EMR Serverless) để thực hiện bảo trì định kỳ cho bảng MOR:

```python
"""
Iceberg Table Maintenance Script: Compaction & Snapshot Expiration
Engine: PySpark 3.4+ / AWS Glue 4.0
"""
from pyspark.sql import SparkSession

def init_spark_iceberg() -> SparkSession:
    """Khởi tạo SparkSession kết nối AWS Glue Catalog và S3 Iceberg."""
    return (
        SparkSession.builder
        .appName("Iceberg-Table-Compaction")
        .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
        .config("spark.sql.catalog.glue_catalog", "org.apache.iceberg.spark.SparkCatalog")
        .config("spark.sql.catalog.glue_catalog.catalog-impl", "org.apache.iceberg.aws.glue.GlueCatalog")
        .config("spark.sql.catalog.glue_catalog.warehouse", "s3://my-lakehouse-bucket/warehouse/")
        .getOrCreate()
    )

def compact_iceberg_table(spark: SparkSession, table_name: str):
    """
    1. Rewrite Data Files: Gộp các file nhỏ và merge các Delete Files vào Data Files.
    2. Rewrite Manifests: Tối ưu hóa cây chỉ mục manifest.
    3. Expire Snapshots: Dọn dẹp metadata và file rác cũ trên S3 quá 7 ngày.
    """
    print(f"[*] Bắt đầu Compaction cho bảng: {table_name}")
    
    # 1. Chạy Compaction: Gom data files về kích thước chuẩn 512MB và nén Delete files
    spark.sql(f"""
        CALL glue_catalog.system.rewrite_data_files(
            table => '{table_name}',
            strategy => 'binpack',
            options => map(
                'target-file-size-bytes', '536870912', -- 512 MB
                'min-input-files', '5',
                'delete-file-threshold', '2'          -- Merge nếu có từ 2 delete files
            )
        )
    """).show(truncate=False)

    # 2. Gom manifest files
    print(f"[*] Đang tối ưu hóa manifest files cho: {table_name}")
    spark.sql(f"""
        CALL glue_catalog.system.rewrite_manifests(
            table => '{table_name}'
        )
    """).show(truncate=False)

    # 3. Xóa các Snapshot cũ hơn 7 ngày để giải phóng dung lượng lưu trữ trên S3
    print(f"[*] Dọn dẹp Snapshot cũ cho: {table_name}")
    spark.sql(f"""
        CALL glue_catalog.system.expire_snapshots(
            table => '{table_name}',
            older_than => TIMESTAMP '{spark.sql("SELECT CURRENT_TIMESTAMP() - INTERVAL 7 DAYS").collect()[0][0]}',
            retain_last => 5
        )
    """).show(truncate=False)
    
    print(f"[✓] Hoàn tất chu trình bảo trì bảng: {table_name}")

if __name__ == "__main__":
    spark_session = init_spark_iceberg()
    target_table = "glue_catalog.iceberg_demo_db.orders_mor"
    compact_iceberg_table(spark_session, target_table)
    spark_session.stop()
```

---

# IV. Lesson learned / Tổng kết

Sau nhiều năm vận hành các hệ thống Petabyte-scale Lakehouse trên nền tảng Apache Iceberg và AWS Athena, mình rút ra 4 quy tắc vàng giúp các bạn đưa ra quyết định chuẩn xác giữa COW và MOR:

1. **Nguyên tắc chọn chế độ theo hình mẫu truy cập (Workload Profile):**
   - **Bảng Fact lớn (Khối lượng ghi là Append, Batch Update theo ngày/tuần):** Hãy kiên quyết chọn **Copy-On-Write (COW)**. Tốc độ đọc trên AWS Athena sẽ luôn đạt hiệu năng đỉnh cao, tiết kiệm hàng ngàn USD chi phí quét dữ liệu (Data Scanned), và loại bỏ hoàn toàn gánh nặng phải vận hành cụm compaction phức tạp.
   - **Bảng CDC / Dimension thay đổi liên tục (Ghi streaming hàng ngàn records/giây):** Bắt buộc phải dùng **Merge-On-Read (MOR)**. Nếu các bạn cố dùng COW cho streaming CDC, pipeline ghi sẽ liên tục bị nghẽn (OOM hoặc trễ SLA) vì hiện tượng Write Amplification.
2. **MOR không thể sống thiếu Compaction:** Hãy coi việc chạy Compaction trong kiến trúc MOR giống như việc hút bụi dọn nhà. Nếu bạn thiết lập bảng MOR mà không lên lịch cron job chạy `rewrite_data_files` hàng ngày, chỉ sau 1 tháng số lượng Delete Files sẽ bóp nghẹt Athena, khiến các câu query đơn giản cũng bị time out.
3. **Tận dụng Hidden Partitioning để khoanh vùng Delete Files:** Khi thiết kế bảng Iceberg, hãy sử dụng tính năng phân vùng ẩn (ví dụ: `month(created_at)`). Khi có câu lệnh `DELETE` theo phân vùng, Iceberg chỉ tạo Delete Files trong đúng phân vùng đó. Khi người dùng query dữ liệu tháng gần nhất, Athena sẽ tự động prune các phân vùng cũ và không cần phải đọc các Delete Files không liên quan.
4. **Cẩn trọng với Equality Deletes:** Equality Deletes rất tiện lợi cho Apache Flink, nhưng chi phí đọc của nó cao hơn Position Deletes rất nhiều lần trên các Query Engine như Athena. Nếu có thể, hãy cấu hình engine streaming ghi ra **Position Deletes** hoặc kích hoạt Compaction chuyển đổi Equality Deletes thành Position Deletes càng sớm càng tốt.

Hiểu rõ bản chất COW và MOR giúp chúng ta làm chủ hoàn toàn Apache Iceberg, mang lại trải nghiệm truy vấn siêu tốc cho người dùng cuối mà vẫn giữ cho hệ sinh thái hạ tầng dữ liệu luôn tinh gọn và tiết kiệm chi phí. Chúc các bạn áp dụng thành công vào hệ thống của mình!
