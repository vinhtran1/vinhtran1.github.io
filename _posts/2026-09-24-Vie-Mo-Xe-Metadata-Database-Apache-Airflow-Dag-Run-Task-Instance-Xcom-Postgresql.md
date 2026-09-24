---
title: 'Mổ xẻ Metadata Database Apache Airflow: dag_run, task_instance, xcom và chiến lược tối ưu cơ sở dữ liệu PostgreSQL cho Enterprise'
date: 2026-09-24 11:30:00 +0700
categories: [Data Engineering, Workflow Orchestration]
tags: [Apache Airflow, PostgreSQL, Database Optimization, Metadata, Data Engineering]
keywords: [Apache Airflow, PostgreSQL, Database Optimization, Metadata]
pin: false
image:
  path: /assets/img/posts/2026/mo-xe-metadata-database-apache-airflow-dag-run-task-instance-xcom-postgresql/cover.webp
  alt: 'Sơ đồ kiến trúc Metadata Database của Apache Airflow trên PostgreSQL và các giải pháp tối ưu DB'
---

# I. Dẫn nhập: Đặt vấn đề và Thách thức kỹ thuật

Một buổi sáng thứ Hai đẹp trời, khi các kỹ sư mở giao diện Webserver của Apache Airflow để kiểm tra lịch chạy đầu tuần thì màn hình quay tròn liên tục. Trang Grid View mất tới hơn 40 giây mới hiển thị được cây thư mục task. Đồng thời trên kênh Slack cảnh báo hạ tầng, Airflow Scheduler bắt đầu bắn thông báo lỗi dồn dập:
- `Scheduler heartbeat dropped for 120 seconds`
- `Zombie tasks detected: TaskInstance <TaskInstance: core_etl.aggregate_sales ... [running]> marked as failed.`

Các kỹ sư vội vã kiểm tra cụm Celery Worker và Pod Kubernetes nhưng mọi thứ vẫn hoạt động bình thường, không hề có tình trạng thiếu CPU hay thiếu RAM. Nhưng khi mình mở bảng điều khiển của cơ sở dữ liệu Amazon RDS PostgreSQL — nơi đóng vai trò làm Metadata Database cho Airflow — thì bức tranh thật sự mới lộ diện: **CPU của PostgreSQL chạm đỉnh 99%, hàng chục câu truy vấn `SELECT ... FOR UPDATE SKIP LOCKED` bị nghẽn (Lock contention), và dung lượng đĩa đã phình to vượt mốc 350 GB!**

```
[PostgreSQL Performance Insights - Airflow Metadata DB]
Engine Load (vCPU) : [████████████████████████████████] 99.4% (Max: 8 vCPUs)
Active Connections : 285 / 300 Max Connections
Dead Tuples Count  : 42,800,000 dead tuples on 'task_instance' table!
Top Culprit Query  : SELECT * FROM task_instance WHERE state IN ('scheduled') FOR UPDATE SKIP LOCKED
Disk Space Usage   : 362 GB (xcom table accounts for 248 GB!)
```

Tại sao một hệ thống điều phối tác vụ (Workflow Orchestrator) như Airflow lại có thể làm "nát" cả một cụm PostgreSQL cỡ lớn như vậy? Rất nhiều bạn khi bắt đầu với Airflow thường chỉ tập trung vào việc viết mã Python cho DAGs, định nghĩa các Operator và Sensor, mà quên mất rằng: **Bản thân các tiến trình Airflow (Scheduler, Webserver, Triggerer, Worker) là các ứng dụng hoàn toàn phi trạng thái (Stateless). Toàn bộ trạng thái của hệ thống, vòng đời của từng task, lịch trình kích hoạt và dữ liệu trao đổi giữa các task đều được lưu trữ tập trung tại Metadata Database.**

Sau 1 đến 2 năm vận hành trong môi trường doanh nghiệp với hàng trăm DAGs chạy mỗi phút:
1. Bảng `task_instance` phình to lên tới hàng chục triệu dòng với hàng triệu **Dead Tuples** sinh ra do các thao tác `UPDATE` trạng thái dồn dập.
2. Bảng `xcom` bị các developer lạm dụng làm "kho chứa tạm" cho các payload JSON, dictionary lớn hoặc thậm chí cả Pandas DataFrames.
3. Tiến trình dọn rác tự động (`autovacuum`) của PostgreSQL không thể theo kịp tốc độ sinh rác, khiến các câu truy vấn lập lịch cốt lõi của Scheduler rơi vào trạng thái quét toàn bảng (Sequential Scan) chậm chạp.

Trong bài viết chuyên sâu này, mình sẽ cùng các bạn mổ xẻ cấu trúc lược đồ quan hệ (Entity-Relationship) của Airflow Metadata Database trên PostgreSQL, phân tích cơ chế then chốt của Scheduler (Critical Section), cung cấp bộ SQL chẩn đoán bloat và dead tuples, đồng thời xây dựng một chiến lược tối ưu hóa toàn diện: từ tinh chỉnh `postgresql.conf`, xây dựng Custom XCom Backend lưu trữ S3, cho đến thiết lập Maintenance DAG tự động hóa việc dọn dẹp hệ thống.

---

# II. Kiến trúc / Nguyên lý cốt lõi

## 1. Bản đồ thực thể (Entity-Relationship Diagram) của Airflow DB

Để tối ưu hóa được cơ sở dữ liệu, trước tiên chúng ta phải nắm rõ những bảng nào đang âm thầm hoạt động dưới nắp ca-pô của Airflow. Dưới đây là lược đồ quan hệ giữa các bảng quan trọng nhất:

```
+-----------------------------------------------------------------------------------+
|                                 AIRFLOW METADATA DB                               |
+-----------------------------------------------------------------------------------+

     +-------------------+
     |        dag        |
     +-------------------+
     | dag_id (PK)       |
     | is_paused         |
     | fileloc, hash     |
     +-------------------+
               | 1
               |
               | N
               v
     +-------------------+                          +---------------------------+
     |      dag_run      |                          |            job            |
     +-------------------+                          +---------------------------+
     | id (PK)           |                          | id (PK)                   |
     | dag_id, run_id    |                          | job_type (SchedulerJob)   |
     | state (running..) |                          | latest_heartbeat          |
     | execution_date    |                          | executor_class            |
     +-------------------+                          +---------------------------+
               | 1
               |
               | N
               v
     +--------------------------------------------------------------------------+
     |                              task_instance                               |
     +--------------------------------------------------------------------------+
     | (task_id, dag_id, run_id, map_index) [COMPOSITE PRIMARY KEY]            |
     | state ('scheduled', 'queued', 'running', 'success', 'failed'...)         |
     | start_date, end_date, duration, try_number, max_tries                     |
     | hostname, unixname, pool, queue, priority_weight                         |
     +--------------------------------------------------------------------------+
          | 1                                               | 1
          |                                                 |
          | N                                               | N
          v                                                 v
     +-------------------+                          +---------------------------+
     |       xcom        |                          |      task_reschedule      |
     +-------------------+                          +---------------------------+
     | (dag_id, task_id, |                          | (dag_id, task_id, run_id, |
     |  run_id, map_idx, |                          |  reschedule_date)         |
     |  key) [PK]        |                          | duration, start_date      |
     | value (BYTEA/JSON)|                          +---------------------------+
     +-------------------+
```

### Các bảng dữ liệu cốt lõi:
1. **`dag`**: Chứa thông tin đăng ký của các workflow, bao gồm ID, file path trên ổ đĩa, cờ bật/tắt (`is_paused`), và hàm băm mã nguồn để Scheduler phát hiện khi nào file Python bị thay đổi.
2. **`dag_run`**: Mỗi lần DAG được kích hoạt (dù là do lịch trình cron, trigger bằng tay, hoặc dataset-driven), một bản ghi mới được tạo ra ở đây với các trạng thái: `queued`, `running`, `success`, `failed`.
3. **`task_instance` (TI)**: Đây là **bảng có khối lượng ghi và kích thước lớn nhất trong toàn bộ hệ thống**. Bảng này sở hữu khóa chính phức hợp gồm 4 trường: `(task_id, dag_id, run_id, map_index)`. Mọi thông tin về vòng đời thực thi, worker nhận việc, thời gian chạy, số lần thử lại đều tập trung tại đây.
4. **`xcom` (Cross-Communication)**: Cơ chế trao đổi siêu dữ liệu giữa các task. Dữ liệu được lưu trữ dạng cặp Key-Value. Trong PostgreSQL, cột `value` được lưu dưới dạng `bytea` (hoặc JSON trong các phiên bản mới).
5. **`job`**: Quản lý nhịp tim (heartbeat) của các tiến trình nền. Mỗi khi Scheduler hoặc Triggerer chạy, nó liên tục cập nhật trường `latest_heartbeat` vào bảng này để các tiến trình khác biết nó còn sống.

## 2. Điểm nghẽn truy vấn của Scheduler: Critical Section

Trái tim của Apache Airflow là vòng lặp lập lịch (`SchedulerLoop`). Cứ sau mỗi vài phần trăm giây, Scheduler sẽ bước vào một đoạn mã quan trọng gọi là **Critical Section** (`_critical_section_enqueue_task_instances`). Nhiệm vụ của nó là tìm kiếm các task instance đang sẵn sàng chạy để đưa vào hàng đợi của Executor:

```sql
-- Câu truy vấn then chốt được Scheduler gọi liên tục
SELECT task_instance.task_id, task_instance.dag_id, task_instance.run_id
FROM task_instance
JOIN dag_run ON task_instance.dag_id = dag_run.dag_id AND task_instance.run_id = dag_run.run_id
WHERE task_instance.state = 'scheduled'
  AND dag_run.state = 'running'
ORDER BY task_instance.priority_weight DESC, task_instance.execution_date ASC
LIMIT 32
FOR UPDATE OF task_instance SKIP LOCKED;
```

### Cơ chế `FOR UPDATE SKIP LOCKED`
Mệnh đề `FOR UPDATE SKIP LOCKED` trong PostgreSQL là một tính năng cực kỳ mạnh mẽ: nó cho phép nhiều tiến trình Scheduler cùng chạy song song (Multi-Scheduler HA) mà không khóa lẫn nhau. Tiến trình A sẽ khóa các dòng nó đang xử lý, và tiến trình B sẽ tự động bỏ qua (skip) những dòng đã bị khóa để nhặt các dòng tiếp theo.

Tuy nhiên, **nếu bảng `task_instance` bị phình to (bloat) và chứa hàng chục triệu dead tuples**:
- PostgreSQL Optimizer sẽ không thể sử dụng index hiệu quả.
- Câu truy vấn buộc phải thực hiện quét chỉ mục rộng (Index Range Scan) hoặc quét toàn bảng (Seq Scan) qua hàng triệu trang đĩa (pages) chỉ chứa dead tuples.
- Thời gian thực thi của câu truy vấn vọt từ **2ms** lên tới **1,500ms - 3,000ms**.
- Hậu quả: Scheduler bị nghẽn (starvation), không kịp cập nhật heartbeat của chính nó lên bảng `job`, dẫn đến việc các Scheduler khác tưởng rằng đồng nghiệp đã chết và liên tục kích hoạt cơ chế dọn dẹp nhầm!

## 3. Cạm bẫy XCom Bloat & Dead Tuples

### Cạm bẫy 1: XCom Bloat và cơ chế TOAST của PostgreSQL
Khi một developer thực hiện `return large_dict` hoặc `ti.xcom_push(key='data', value=df.to_json())`, một chuỗi JSON có thể nặng tới 10MB – 50MB. Trong PostgreSQL, một trang đĩa (page) có kích thước cố định là 8KB. Bất kỳ giá trị nào vượt quá khoảng 2KB sẽ được đưa vào cơ chế **TOAST (The Oversized-Attribute Storage Technique)** — tức là cắt nhỏ dữ liệu và lưu vào một bảng phụ TOAST riêng biệt.

Hậu quả:
- Dung lượng bảng `xcom` và bảng TOAST liên kết phình to không kiểm soát.
- Mỗi lần Scheduler hoặc Webserver cần hiển thị trang chi tiết của Task, nó phải thực hiện phép ghép (join) và giải nén dữ liệu TOAST khổng lồ, làm cạn kiệt bộ nhớ đệm `shared_buffers` của PostgreSQL.

### Cạm bẫy 2: Cơn bão Dead Tuples trên bảng `task_instance`
Trong kiến trúc MVCC (Multi-Version Concurrency Control) của PostgreSQL, một câu lệnh `UPDATE` thực chất là một thao tác `INSERT` một bản ghi mới và đánh dấu bản ghi cũ là "chết" (Dead Tuple).

Hãy làm một bài toán nhỏ: Trong vòng đời của một TaskInstance bình thường, trạng thái của nó sẽ được cập nhật ít nhất 5 lần:
$$\text{None} \longrightarrow \text{scheduled} \longrightarrow \text{queued} \longrightarrow \text{running} \longrightarrow \text{success}$$

Nếu hệ thống của bạn xử lý **20,000 tasks mỗi ngày**:
$$\text{Số lượng Dead Tuples sinh ra} = 20,000 \times 5 = 100,000 \text{ dead tuples/ngày}$$
Sau 1 tháng, bảng `task_instance` tích lũy hơn **3,000,000 dead tuples**. Nếu cấu hình `autovacuum` mặc định quá bảo thủ (chỉ kích hoạt khi số lượng dead tuples vượt quá 20% dung lượng bảng), bảng sẽ phải chờ tới khi phình to khổng lồ mới được dọn, gây ra hiện tượng phân mảnh đĩa nặng nề.

---

# III. Cài đặt / Hands-on code: Hiện thực & Tối ưu thực chiến

## 1. Bộ SQL Diagnostics chẩn đoán bệnh cho Airflow Database

Dưới đây là bộ câu lệnh SQL thực chiến mà các bạn có thể chạy trực tiếp trên PostgreSQL để kiểm tra ngay lập tức tình trạng sức khỏe của Metadata DB:

```sql
-- 1. Đo lường dung lượng thực tế và tỷ lệ Dead Tuples của từng bảng Airflow
SELECT 
    schemaname,
    relname AS table_name,
    n_live_tup AS live_tuples,
    n_dead_tup AS dead_tuples,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_tuple_ratio_pct,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size_including_indexes,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- 2. Tìm danh sách Top 10 bản ghi XCom lớn nhất đang ngốn đĩa
SELECT 
    dag_id,
    task_id,
    run_id,
    key,
    pg_size_pretty(octet_length(value)) AS raw_byte_size,
    timestamp
FROM xcom
ORDER BY octet_length(value) DESC
LIMIT 10;

-- 3. Kiểm tra độ trễ xếp hàng (Queue Latency) của các Task gần đây
SELECT 
    dag_id,
    task_id,
    state,
    queued_dttm,
    start_date,
    ROUND(EXTRACT(EPOCH FROM (start_date - queued_dttm))::numeric, 2) AS queue_wait_seconds
FROM task_instance
WHERE queued_dttm IS NOT NULL AND start_date IS NOT NULL
ORDER BY queued_dttm DESC
LIMIT 15;
```

## 2. Tối ưu hóa cấu hình PostgreSQL cho Airflow

Để cơ sở dữ liệu PostgreSQL có thể đáp ứng mượt mà tần suất ghi và cập nhật cực lớn từ Airflow, các bạn cần áp dụng các thông số chuyên dụng sau vào file cấu hình `postgresql.conf` (hoặc Parameter Group trên AWS RDS):

```ini
# ==============================================================================
# POSTGRESQL TUNING FOR APACHE AIRFLOW METADATA DATABASE
# ==============================================================================

# 1. Tối ưu hóa Autovacuum cho bảng có tỷ lệ UPDATE cực cao
# Mặc định scale_factor là 0.2 (20%). Chúng ta hạ xuống 0.05 (5%) đối với bảng Airflow
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_cost_limit = 2000      # Tăng hạn mức I/O để dọn dẹp nhanh hơn
autovacuum_vacuum_cost_delay = 2ms       # Giảm độ trễ giữa các lần quét dọn
autovacuum_max_workers = 4

# 2. Bộ nhớ đệm và cấp phát
shared_buffers = 4GB                     # Cấp khoảng 25% tổng RAM của máy chủ DB
effective_cache_size = 12GB              # Cấp khoảng 75% tổng RAM
work_mem = 64MB                          # Đủ cho các phép SORT của Scheduler
maintenance_work_mem = 512MB             # Tăng tốc độ cho VACUUM và REINDEX

# 3. Quản lý kết nối và Transaction Lock
max_connections = 300
statement_timeout = 30000                # Hủy truy vấn nếu chạy quá 30 giây (trừ vacuum)
idle_in_transaction_session_timeout = 60000 # Hủy session idle quá 1 phút tránh giữ lock

# 4. Ghi trước nhật ký (WAL)
wal_buffers = 16MB
checkpoint_completion_target = 0.9
max_wal_size = 8GB
min_wal_size = 1GB
```

> **Lưu ý sống còn về Connection Pooling (PgBouncer)**: Nếu sử dụng PgBouncer đứng trước Airflow PostgreSQL, các bạn **BẮT BUỘC** phải cấu hình chế độ `pool_mode = transaction`. Tuy nhiên, vì SQLAlchemy trong Airflow có thể sử dụng `PREPARE statement`, các bạn cần vô hiệu hóa prepared statements phía client hoặc đảm bảo `server_reset_query = DISCARD ALL` để tránh lỗi `prepared statement does not exist`.

## 3. Triển khai Custom XCom Backend lưu trữ trên Amazon S3

Để triệt tiêu vĩnh viễn nguy cơ phình to của bảng `xcom`, giải pháp chuẩn mực nhất là triển khai **Custom XCom Backend**. Cơ chế này sẽ chặn mọi giá trị XCom: nếu payload vượt quá ngưỡng quy định (ví dụ 1KB), nó sẽ ghi file thẳng lên Amazon S3 và chỉ lưu đường dẫn URI vào PostgreSQL!

```python
"""Custom S3 XCom Backend for Apache Airflow."""
import json
import uuid
from typing import Any
import boto3
from airflow.models.xcom import BaseXCom

S3_BUCKET = "enterprise-airflow-xcom-storage"
S3_PREFIX = "xcom_payloads"
PAYLOAD_SIZE_THRESHOLD_BYTES = 1024  # 1 KB

class S3CustomXComBackend(BaseXCom):
    """
    XCom Backend tùy chỉnh: Lưu payload nhỏ trong Postgres, 
    đẩy payload lớn lên S3 để chống XCom Table Bloat.
    """
    @staticmethod
    def serialize_value(value: Any, **kwargs) -> Any:
        # Chuyển đổi dữ liệu sang dạng chuỗi JSON
        serialized_str = json.dumps(value, default=str)
        payload_bytes = serialized_str.encode('utf-8')

        # Nếu dữ liệu nhỏ hơn 1KB, lưu bình thường vào Postgres
        if len(payload_bytes) <= PAYLOAD_SIZE_THRESHOLD_BYTES:
            return BaseXCom.serialize_value(value)

        # Ngược lại, nạp lên Amazon S3
        s3_client = boto3.client('s3')
        object_key = f"{S3_PREFIX}/{uuid.uuid4().hex}.json"
        
        s3_client.put_object(
            Bucket=S3_BUCKET,
            Key=object_key,
            Body=payload_bytes,
            ContentType='application/json'
        )

        # Chỉ lưu con trỏ S3 URI vào metadata database
        pointer = {"__s3_xcom_pointer__": True, "s3_uri": f"s3://{S3_BUCKET}/{object_key}"}
        return BaseXCom.serialize_value(pointer)

    @staticmethod
    def deserialize_value(result: Any) -> Any:
        deserialized = BaseXCom.deserialize_value(result)
        
        # Kiểm tra xem đây có phải là con trỏ S3 hay không
        if isinstance(deserialized, dict) and deserialized.get("__s3_xcom_pointer__"):
            s3_uri = deserialized["s3_uri"]
            # Tách bucket và key từ URI
            bucket, key = s3_uri.replace("s3://", "").split("/", 1)
            
            s3_client = boto3.client('s3')
            response = s3_client.get_object(Bucket=bucket, Key=key)
            content = response['Body'].read().decode('utf-8')
            return json.loads(content)

        return deserialized
```

Để kích hoạt module này trong Airflow, các bạn chỉ cần thêm biến môi trường vào container:
```bash
AIRFLOW__CORE__XCOM_BACKEND=custom_backends.s3_xcom.S3CustomXComBackend
```

## 4. Xây dựng Maintenance DAG: Tự động hóa `airflow db clean`

Từ Airflow 2.3+, câu lệnh `airflow db clean` đã được tích hợp sẵn để dọn dẹp các bản ghi cũ của các bảng `dag_run`, `task_instance`, `log`, `xcom`. Dưới đây là DAG bảo trì định kỳ chạy vào 1:00 AM sáng Chủ Nhật hàng tuần:

```python
"""Automated database maintenance DAG for Airflow metadata tables."""
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator

DEFAULT_ARGS = {
    'owner': 'data-infrastructure',
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    dag_id='airflow_metadata_db_maintenance',
    default_args=DEFAULT_ARGS,
    start_date=datetime(2026, 9, 1),
    schedule_interval='0 1 * * 0',  # 01:00 AM mỗi Chủ Nhật
    catchup=False,
    max_active_runs=1,
    tags=['maintenance', 'database', 'cleanup'],
) as dag:

    # Dọn dẹp dữ liệu cũ hơn 60 ngày theo từng đợt để tránh Table Lock kéo dài
    clean_metadata_tables = BashOperator(
        task_id='clean_historical_metadata',
        bash_command="""
            echo "Bắt đầu dọn dẹp Metadata Database cũ hơn 60 ngày..."
            airflow db clean \
                --clean-before-timestamp $(date -d "60 days ago" +%Y-%m-%d) \
                --skip-archive \
                --yes
        """,
    )

    # Chạy VACUUM ANALYZE trên PostgreSQL để cập nhật lại query planner
    vacuum_analyze_db = BashOperator(
        task_id='vacuum_analyze_tables',
        bash_command="""
            echo "Thực hiện VACUUM ANALYZE các bảng trọng yếu..."
            PGPASSWORD=$AIRFLOW_DB_PASS psql -h $AIRFLOW_DB_HOST -U $AIRFLOW_DB_USER -d $AIRFLOW_DB_NAME -c \
            "VACUUM (ANALYZE, VERBOSE) task_instance; VACUUM (ANALYZE, VERBOSE) dag_run; VACUUM (ANALYZE, VERBOSE) xcom;"
        """,
    )

    clean_metadata_tables >> vacuum_analyze_db
```

---

# IV. Lesson learned: Tổng kết & Best Practices

Quản trị cơ sở dữ liệu metadata của Airflow đòi hỏi tư duy của một Database Administrator kết hợp với một Data Engineer. Dưới đây là 5 bài học sống còn mình đúc kết được:

1. **Tuyệt đối không sử dụng XCom làm Data Bus**:
   - XCom chỉ được sinh ra để trao đổi các thông điệp gọn nhẹ: số lượng dòng đã xử lý, đường dẫn tệp S3, hoặc cờ trạng thái logic (`is_completed=True`).
   - Mọi dữ liệu lớn hơn vài Kilobytes bắt buộc phải được lưu trữ trên Object Storage (Amazon S3 hoặc Google Cloud Storage).

2. **Kích hoạt Custom XCom Backend ngay từ giai đoạn thiết kế**:
   - Trong môi trường doanh nghiệp đông developer, bạn không thể kiểm soát 100% mã nguồn từng người viết. Việc kích hoạt `S3CustomXComBackend` sẽ hoạt động như một tấm lưới an toàn (safety net), tự động đánh chặn và giải tỏa áp lực đĩa cho PostgreSQL.

3. **Chủ động thiết lập Retention Policy (30 – 90 ngày)**:
   - Đừng lưu trữ lịch sử chạy task mãi mãi trong Metadata Database. Lịch sử của 6 tháng trước không giúp ích gì cho Scheduler hiện tại mà chỉ làm chậm hệ thống. Hãy dọn dẹp định kỳ và đẩy các số liệu báo cáo thời gian chạy sang Data Warehouse riêng (như BigQuery hoặc Snowflake) để phục vụ phân tích SLA lâu dài.

4. **Tách biệt hoàn toàn Database của Airflow**:
   - Không bao giờ dùng chung PostgreSQL instance của Airflow với cơ sở dữ liệu nghiệp vụ của ứng dụng web hay dịch vụ backend khác. Tần suất khóa dòng và quét liên tục của Scheduler sẽ ảnh hưởng trực tiếp tới độ trễ (latency) của các ứng dụng chung nhà.

5. **Giám sát chặt chẽ chỉ số PostgreSQL qua APM**:
   - Thiết lập cảnh báo tự động khi:
     - `Dead Tuple Ratio > 20%` trên bảng `task_instance`.
     - Tỷ lệ Cache Hit Ratio của PostgreSQL rơi xuống dưới 98%.
     - Thời gian thực thi của câu truy vấn `SELECT ... FOR UPDATE SKIP LOCKED` vượt quá 50ms.
