---
title: 'Xây dựng Lean Data Lakehouse giá rẻ: Ingestion và Transformation với DuckDB, PyIceberg, AWS S3 và Airflow'
date: 2026-09-24 13:00:00 +0700
categories: [Data Engineering, Lakehouse]
tags: [Lean Lakehouse, DuckDB, PyIceberg, AWS S3, Apache Airflow, Python, Data Architecture]
keywords: [Lean Lakehouse, DuckDB, PyIceberg, AWS S3, Apache Airflow]
pin: false
image:
  path: /assets/img/posts/2026/lean-data-lakehouse-gia-re-duckdb-pyiceberg-s3-airflow/cover.webp
  alt: 'Kiến trúc Lean Data Lakehouse giá rẻ với DuckDB, PyIceberg, AWS S3 và Apache Airflow'
---

# I. Dẫn nhập

Chào các bạn, trong những năm gần đây, "Modern Data Stack" trở thành một trong những từ khóa thời thượng nhất trong giới kỹ sư dữ liệu. Mọi hội thảo công nghệ đều nói về Snowflake, Databricks, cụm Apache Spark trên Amazon EMR, dbt Cloud và Fivetran. Những giải pháp này quả thực vô cùng mạnh mẽ, có khả năng xử lý quy mô hàng chục Petabytes dữ liệu một cách mượt mà.

Thế nhưng, có một sự thật trớ trêu mà ít ai nhắc tới: **Chi phí duy trì những hệ thống này là một "cơn ác mộng" đối với các doanh nghiệp vừa và nhỏ (SMBs) hoặc các công ty khởi nghiệp (Startups)**.

Hãy thử làm một bài toán thực tế:
- Một doanh nghiệp thương mại điện tử tầm trung sở hữu khoảng 500 GB đến 2 TB dữ liệu, mỗi ngày phát sinh thêm chừng 2 đến 5 triệu dòng sự kiện (khoảng vài GB dữ liệu nén).
- Nếu dựng một cụm Amazon EMR hoặc Databricks chạy 24/7 để phục vụ Ingestion và ELT, hóa đơn cloud hàng tháng dễ dàng vượt mốc $1,500 – $3,000 USD.
- Nếu dùng Snowflake, chi phí Credit cho các Warehouse chạy liên tục để nạp dữ liệu vi mô (micro-batches) cũng tiêu tốn ngân sách không hề nhỏ.

Chúng ta đang dùng "dao mổ trâu để giết gà"! Phần lớn các bài toán phân tích kinh doanh ở quy mô dưới vài Terabytes hoàn toàn không cần đến năng lực tính toán phân tán (Distributed Computing) phức tạp của hàng chục máy chủ Spark.

Đó là lý do kiến trúc **Lean Data Lakehouse (Lakehouse tinh gọn)** ra đời. Triết lý của Lean Lakehouse là: **Tận dụng tối đa sức mạnh của điện toán cục bộ đơn máy (Single-node Vectorized Engine), kết hợp với định dạng bảng mở thế hệ mới và lưu trữ đám mây giá rẻ để cắt giảm chi phí hạ tầng tới hơn 90%**.

Bộ tứ công nghệ tạo nên kỳ tích này bao gồm:
1. **DuckDB:** "SQLite của kỷ nguyên OLAP", engine phân tích dữ liệu dạng cột siêu tốc chạy trực tiếp trong tiến trình (in-process).
2. **PyIceberg (phiên bản 0.9.0 trở lên):** Thư viện Python thuần túy cho phép tương tác và Upsert/Merge vào bảng **Apache Iceberg** mà không cần cài đặt máy ảo Java (JVM) hay cụm Spark cồng kềnh.
3. **Amazon S3:** Tầng lưu trữ hướng đối tượng với độ bền 99.999999999% và chi phí cực rẻ ($0.023 / GB / tháng).
4. **Apache Airflow:** Trình điều phối (Orchestrator) quen thuộc điều khiển luồng công việc.

Trong bài viết này, mình sẽ cùng các bạn mổ xẻ kiến trúc Lean Lakehouse, cách DuckDB truy vấn trực tiếp file S3 qua bộ nhớ RAM, cơ chế Upsert không cần Spark bằng PyIceberg, và hoàn thiện một pipeline nạp dữ liệu thực chiến với tổng chi phí vận hành chỉ vài chục USD mỗi tháng.

---

# II. Kiến trúc / Nguyên lý

Trước khi bắt tay vào triển khai, chúng ta hãy cùng nhìn vào bức tranh kiến trúc tổng thể của Lean Data Lakehouse:

```
+---------------------------------------------------------------------------------------------------+
|                                  LEAN DATA LAKEHOUSE ARCHITECTURE                                 |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|    +-----------------------------+                                                                |
|    | External Sources (API, DB)  |                                                                |
|    +--------------+--------------+                                                                |
|                   |                                                                               |
|                   | 1. Extract Batch                                                              |
|                   v                                                                               |
|    +-------------------------------------------------------------------------------------+        |
|    |                           APACHE AIRFLOW ORCHESTRATOR                               |        |
|    |                                                                                     |        |
|    |  +---------------------------+   2. Query S3 HTTPFS   +--------------------------+  |        |
|    |  |  Task 1: S3 Raw Ingest    |----------------------->| Task 2: DuckDB Transform |  |        |
|    |  | (Parquet Bronze Storage)  |                        | (Vectorized in-memory)   |  |        |
|    |  +---------------------------+                        +------------+-------------+  |        |
|    |                                                                    |                |        |
|    |                                                       3. Zero-Copy | Arrow Table    |        |
|    |                                                                    v                |        |
|    |                                                       +--------------------------+  |        |
|    |                                                       | Task 3: PyIceberg Upsert |  |        |
|    |                                                       | (Commit Snapshot to S3)  |  |        |
|    |                                                       +------------+-------------+  |        |
|    +--------------------------------------------------------------------|----------------+        |
|                                                                         |                         |
|                                4. ACID Commit                           |                         |
|                   +-----------------------------------------------------+                         |
|                   |                                                                               |
|                   v                                                                               |
|    +-----------------------------+         +-------------------------------+                      |
|    |   AWS Glue Data Catalog     |<------->|       Amazon S3 Warehouse     |                      |
|    |   (Central Metadata Lock)   |         |   (Iceberg Metadata & Data)   |                      |
|    +-----------------------------+         +-------------------------------+                      |
|                   |                                        |                                      |
|                   +-------------------+--------------------+                                      |
|                                       |                                                           |
|                                       v                                                           |
|                        +-----------------------------+                                            |
|                        |   Ad-hoc Analytics / BI     |                                            |
|                        |   (DuckDB / Athena / Superset)                                           |
|                        +-----------------------------+                                            |
+---------------------------------------------------------------------------------------------------+
```

### 1. Cuộc cách mạng DuckDB trên Cloud Object Storage
DuckDB thường được ví như "SQLite dành cho phân tích", nhưng sức mạnh thực sự của nó vượt xa hình dung của nhiều người. 

Khác với các công cụ như Pandas phải nạp toàn bộ dữ liệu vào RAM dưới dạng đối tượng Python chậm chạp, DuckDB được xây dựng bằng C++ với kiến trúc xử lý dạng cột (Columnar-vectorized Execution Engine). Nó hỗ trợ tính toán đa luồng (Multi-threading) và tính năng **Out-of-core Processing** — nghĩa là nếu tập dữ liệu của bạn lớn hơn dung lượng RAM vật lý, DuckDB sẽ tự động chia nhỏ và tràn dữ liệu xuống đĩa đệm một cách thông minh mà không bao giờ bị lỗi `MemoryError` hay `OOM crash`.

Đặc biệt, thông qua tiện ích mở rộng **`httpfs`** và **`aws`**, DuckDB có thể gửi các truy vấn SQL trực tiếp lên Amazon S3:
- Tự động tận dụng tính năng HTTP Range Requests để chỉ đọc đúng các byte dữ liệu cần thiết từ header và footer của file Parquet.
- Bỏ qua các khối dữ liệu không khớp điều kiện lọc (Projection & Predicate Pushdown).
- Tốc độ quét và lọc hàng triệu dòng dữ liệu trên S3 chỉ diễn ra trong vòng vài trăm mili-giây ngay trên một máy ảo tiêu chuẩn 2 vCPU!

### 2. Bước ngoặt PyIceberg 0.9.0: Thoát ly khỏi JVM và Apache Spark
Trong nhiều năm, nếu muốn ghi dữ liệu (Write / Upsert / Merge) vào bảng Apache Iceberg, các kỹ sư bắt buộc phải phụ thuộc vào cụm tính toán Java / Scala như Apache Spark, Apache Flink hoặc Trino. Điều này đồng nghĩa với việc bạn phải dựng và quản lý cả một cụm máy chủ phân tán cồng kềnh chỉ để cập nhật vài trăm nghìn dòng dữ liệu.

Kể từ phiên bản **PyIceberg 0.9.0**, cộng đồng mã nguồn mở đã tạo nên một bước ngoặt vĩ đại: **Hỗ trợ ghi và cập nhật trực tiếp dữ liệu vào Iceberg bằng Python thông qua Apache Arrow**.

Nhờ cơ chế chia sẻ bộ nhớ **Zero-Copy** giữa DuckDB và PyArrow:
$$\text{DuckDB Query} \xrightarrow[\text{Zero-Copy}]{\text{Arrow Table}} \text{PyIceberg} \xrightarrow{\text{Commit}} \text{S3 Iceberg Table}$$
Toàn bộ quá trình từ truy vấn, làm sạch dữ liệu cho tới commit snapshot ACID vào bảng Iceberg diễn ra hoàn toàn trong bộ nhớ RAM của một Python process duy nhất. Dung lượng bộ nhớ tiêu thụ cực kỳ tiết kiệm, có thể chạy mượt mà ngay trên một Airflow Worker container khiêm tốn.

### 3. Vấn đề Concurrency và Locking trên Object Storage S3
Amazon S3 là một Object Storage thuần túy, không hỗ trợ cơ chế khóa tệp (File Lock) hay chuẩn POSIX. Nếu có 2 Airflow Task cùng cố gắng commit một Snapshot mới vào cùng một bảng Iceberg tại cùng một thời điểm, dữ liệu có thể bị xung đột ghi đè.

Giải pháp cho Lean Lakehouse là sử dụng **AWS Glue Data Catalog** làm Catalog trung tâm. 
- AWS Glue Catalog cung cấp cơ chế khóa lạc quan (Optimistic Concurrency Control - OCC) cấp bảng.
- Khi PyIceberg thực hiện commit snapshot, nó sẽ kiểm tra phiên bản metadata hiện tại trên Glue Catalog. Nếu phát hiện có tiến trình khác đã commit trước, PyIceberg sẽ tự động thử lại (Retry) với Snapshot mới mà không làm hỏng tính toàn vẹn của dữ liệu.

---

# III. Cài đặt / Hands-on code

Bây giờ, mình sẽ cùng các bạn xây dựng trọn vẹn kịch bản Lean Data Lakehouse: sử dụng DuckDB để tổng hợp dữ liệu thô từ S3 Bronze, chuyển giao qua Arrow, và dùng PyIceberg để Upsert vào S3 Silver.

### 1. Cài đặt Môi trường Python

Trong môi trường ảo của bạn hoặc Dockerfile của Airflow, cài đặt các thư viện cần thiết:

```bash
pip install "duckdb>=1.0.0" "pyiceberg[glue,s3fs]>=0.9.0" "pyarrow>=15.0.0" boto3
```

### 2. Kịch bản DuckDB Transformation (`transform_duckdb.py`)

Kịch bản này sử dụng DuckDB để quét trực tiếp các file Parquet thô trên S3, làm sạch và tổng hợp dữ liệu giao dịch của khách hàng, sau đó xuất ra bảng `pyarrow.Table`:

```python
"""
Module: transform_duckdb.py
Nhiệm vụ: Truy vấn dữ liệu thô trên S3 bằng DuckDB và xuất ra Apache Arrow Table
"""
import os
import duckdb
import pyarrow as pa

def run_duckdb_transformation(
    s3_raw_path: str, 
    aws_region: str = "ap-southeast-1"
) -> pa.Table:
    """
    Truy vấn phân tích trực tiếp file S3 Parquet và trả về PyArrow Table
    """
    print(f"[*] Khởi tạo kết nối DuckDB in-process...")
    con = duckdb.connect(database=":memory:")
    
    # 1. Cấu hình Extension httpfs và thông tin xác thực AWS
    con.execute("INSTALL httpfs; LOAD httpfs;")
    con.execute(f"SET s3_region = '{aws_region}';")
    con.execute("SET max_memory = '4GB';")
    con.execute("SET preserve_insertion_order = false;")
    
    # Tự động nạp credentials từ biến môi trường hoặc IAM Role
    aws_key = os.getenv("AWS_ACCESS_KEY_ID")
    aws_secret = os.getenv("AWS_SECRET_ACCESS_KEY")
    if aws_key and aws_secret:
        con.execute(f"SET s3_access_key_id = '{aws_key}';")
        con.execute(f"SET s3_secret_access_key = '{aws_secret}';")

    print(f"[*] Đang thực thi truy vấn Vectorized trên: {s3_raw_path}")
    
    # 2. Câu lệnh SQL phân tích và tổng hợp dữ liệu trực tiếp trên S3
    query = f"""
    WITH raw_data AS (
        SELECT 
            CAST(customer_id AS VARCHAR) AS customer_id,
            TRIM(customer_name) AS customer_name,
            LOWER(TRIM(email)) AS email,
            CAST(transaction_amount AS DOUBLE) AS transaction_amount,
            CAST(transaction_time AS TIMESTAMP) AS transaction_time
        FROM read_parquet('{s3_raw_path}')
        WHERE customer_id IS NOT NULL
    )
    SELECT 
        customer_id,
        FIRST(customer_name) AS customer_name,
        FIRST(email) AS email,
        ROUND(SUM(transaction_amount), 2) AS total_spent,
        COUNT(*) AS total_transactions,
        MAX(transaction_time) AS last_active_time,
        CURRENT_DATE AS report_date
    FROM raw_data
    GROUP BY customer_id
    """
    
    # 3. Trích xuất kết quả dưới dạng PyArrow Table (Zero-Copy)
    arrow_table = con.execute(query).arrow()
    print(f"[✓] DuckDB đã xử lý thành công {arrow_table.num_rows} dòng bản ghi!")
    
    con.close()
    return arrow_table

if __name__ == "__main__":
    # Test local mock hoặc chạy trực tiếp với S3
    sample_path = "s3://my-lakehouse-bucket/raw/transactions/*.parquet"
    # df = run_duckdb_transformation(sample_path)
```

### 3. Kịch bản PyIceberg Upsert vào S3 (`upsert_iceberg.py`)

Kịch bản này sử dụng PyIceberg kết nối với AWS Glue Catalog để thực hiện thao tác Upsert (Merge) từ Arrow Table vào bảng Iceberg:

```python
"""
Module: upsert_iceberg.py
Nhiệm vụ: Kết nối AWS Glue Catalog và Upsert dữ liệu từ Arrow Table vào Apache Iceberg
"""
import pyarrow as pa
from pyiceberg.catalog import load_catalog
from pyiceberg.schema import Schema
from pyiceberg.types import (
    StringType, DoubleType, LongType, 
    TimestampType, DateType, NestedField
)
from pyiceberg.partitioning import PartitionSpec, PartitionField
from pyiceberg.transforms import IdentityTransform

CATALOG_NAME = "glue_catalog"
WAREHOUSE_PATH = "s3://my-lakehouse-bucket/iceberg_warehouse/"
DATABASE_NAME = "lean_lakehouse_db"
TABLE_NAME = "customer_summary"

def get_iceberg_catalog():
    """Khởi tạo kết nối tới AWS Glue Catalog"""
    return load_catalog(
        CATALOG_NAME,
        **{
            "type": "glue",
            "s3.region": "ap-southeast-1",
            "warehouse": WAREHOUSE_PATH,
        }
    )

def ensure_iceberg_table_exists(catalog):
    """Đảm bảo bảng Iceberg đã được khởi tạo với Schema chuẩn"""
    full_table_name = f"{DATABASE_NAME}.{TABLE_NAME}"
    
    if catalog.table_exists(full_table_name):
        return catalog.load_table(full_table_name)
    
    print(f"[*] Bảng {full_table_name} chưa tồn tại, đang tiến hành tạo mới...")
    
    # Định nghĩa Schema Iceberg
    schema = Schema(
        NestedField(field_id=1, name="customer_id", field_type=StringType(), required=True),
        NestedField(field_id=2, name="customer_name", field_type=StringType(), required=False),
        NestedField(field_id=3, name="email", field_type=StringType(), required=False),
        NestedField(field_id=4, name="total_spent", field_type=DoubleType(), required=False),
        NestedField(field_id=5, name="total_transactions", field_type=LongType(), required=False),
        NestedField(field_id=6, name="last_active_time", field_type=TimestampType(), required=False),
        NestedField(field_id=7, name="report_date", field_type=DateType(), required=True),
    )
    
    # Cấu hình phân vùng theo report_date
    partition_spec = PartitionSpec(
        PartitionField(source_id=7, field_id=1000, transform=IdentityTransform(), name="report_date")
    )
    
    table = catalog.create_table(
        identifier=full_table_name,
        schema=schema,
        location=f"{WAREHOUSE_PATH}{DATABASE_NAME}/{TABLE_NAME}",
        partition_spec=partition_spec,
        properties={
            "format-version": "2",
            "write.parquet.compression-codec": "zstd",
            "write.target-file-size-bytes": "134217728" # 128 MB
        }
    )
    print(f"[✓] Đã khởi tạo thành công bảng Iceberg: {full_table_name}")
    return table

def upsert_data_to_iceberg(arrow_data: pa.Table):
    """
    Thực hiện thao tác Upsert (Merge) dữ liệu vào bảng Iceberg bằng PyIceberg
    """
    catalog = get_iceberg_catalog()
    table = ensure_iceberg_table_exists(catalog)
    
    print(f"[*] Bắt đầu commit Upsert {arrow_data.num_rows} bản ghi vào bảng Iceberg...")
    
    # PyIceberg 0.9.0+ hỗ trợ phương thức upsert dựa trên join_cols
    table.upsert(
        df=arrow_data,
        join_cols=["customer_id"]
    )
    
    print(f"[✓] Commit thành công Snapshot mới vào S3 Iceberg Table!")

if __name__ == "__main__":
    pass
```

### 4. Tích hợp Hoàn chỉnh trong Apache Airflow DAG

Dưới đây là mã nguồn Airflow DAG điều phối toàn bộ chu trình xử lý:

```python
"""
DAG: lean_lakehouse_pipeline.py
Orchestration: Ingestion thô -> DuckDB Transform -> PyIceberg Upsert
"""
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator

default_args = {
    "owner": "data_engineering",
    "depends_on_past": False,
    "start_date": datetime(2026, 9, 20),
    "email_on_failure": False,
    "retries": 2,
    "retry_delay": timedelta(minutes=3),
}

def etl_lean_lakehouse_task(**context):
    from transform_duckdb import run_duckdb_transformation
    from upsert_iceberg import upsert_data_to_iceberg
    
    execution_date = context["ds"]
    raw_s3_input = f"s3://my-lakehouse-bucket/raw/transactions/date={execution_date}/*.parquet"
    
    # 1. Chạy DuckDB biến đổi dữ liệu sang Arrow
    arrow_table = run_duckdb_transformation(s3_raw_path=raw_s3_input)
    
    if arrow_table.num_rows == 0:
        print("[!] Không có dữ liệu mới, kết thúc task.")
        return
        
    # 2. PyIceberg Upsert vào S3 Iceberg Silver Layer
    upsert_data_to_iceberg(arrow_data=arrow_table)

with DAG(
    dag_id="lean_data_lakehouse_daily_pipeline",
    default_args=default_args,
    schedule_interval="@daily",
    catchup=False,
    max_active_runs=1,
) as dag:

    run_lakehouse_etl = PythonOperator(
        task_id="duckdb_transform_and_pyiceberg_upsert",
        python_callable=etl_lean_lakehouse_task,
        provide_context=True,
    )
```

---

# IV. Lesson learned / Tổng kết

Sau khi triển khai kiến trúc Lean Data Lakehouse vào môi trường production cho nhiều dự án thực tế, mình đúc kết được 5 bài học vô cùng đắt giá:

1. **Hiệu quả cắt giảm chi phí vượt trội (97% Cost Reduction):** Thay vì chi trả từ $1,200 đến $2,500/tháng cho cụm EMR/Snowflake, hệ thống Lean Lakehouse này chỉ tốn của doanh nghiệp khoảng **$35/tháng** (gồm $15 tiền lưu trữ S3 và $20 cho một instance EC2 `t3.large` chạy chung Airflow). Đây là một tỷ suất hoàn vốn đầu tư (ROI) không tưởng đối với bất kỳ startup nào.
2. **Biết rõ giới hạn quy mô của DuckDB:** DuckDB là công cụ đơn máy. Nó hoạt động với tốc độ thần sầu cho các tập dữ liệu dưới **1-2 TB** hoặc các batch job dưới 50 triệu dòng. Nhưng nếu khối lượng dữ liệu trong một lần chạy của bạn vượt quá 100 triệu dòng, hoặc bạn cần thực hiện các phép Shuffle Join phức tạp giữa 3 bảng Fact lớn, đó là lúc bạn nên cân nhắc chuyển giao sang Apache Spark phân tán.
3. **Cấu hình trần bộ nhớ (`max_memory`) cho DuckDB:** Mặc định, DuckDB sẽ cố gắng chiếm dụng tới 80% RAM vật lý của máy chủ. Nếu các bạn chạy DuckDB bên trong cùng container với Airflow Worker, nó có thể khiến Linux OOM Killer "trảm" luôn tiến trình Airflow. Luôn đặt cấu hình trần rõ ràng: `SET max_memory = '4GB';` để bảo vệ sự an toàn của toàn bộ hệ thống.
4. **Luôn sử dụng AWS Glue Data Catalog làm mỏ neo:** Tuyệt đối không lưu metadata của Iceberg dưới dạng file SQLite cục bộ trên máy chạy Airflow. Việc sử dụng AWS Glue Data Catalog giúp bảng Iceberg của bạn có thể được truy vấn đồng thời từ bất kỳ công cụ nào khác như AWS Athena, Presto hay Amazon EMR một cách liền mạch.
5. **Kiểm soát kích thước file Iceberg (`write.target-file-size-bytes`):** Khi PyIceberg ghi dữ liệu ra S3, hãy đặt kích thước file mục tiêu ở mức từ 128 MB đến 256 MB. Đừng để file quá nhỏ (vài MB) gây phân mảnh Small Files, cũng không nên để file quá lớn (1 GB) làm giảm hiệu quả xử lý song song của các query engine sau này.

Hy vọng bài viết này đã mang đến cho các bạn một góc nhìn mới mẻ và thực tế về việc xây dựng nền tảng dữ liệu hiện đại: Không nhất thiết phải tốn kém hàng đống tiền cho các cụm máy chủ cồng kềnh mới có thể sở hữu một Data Lakehouse chuẩn mực!
