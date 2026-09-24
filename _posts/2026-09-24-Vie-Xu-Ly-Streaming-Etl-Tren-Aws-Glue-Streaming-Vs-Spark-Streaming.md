---
title: 'Xử lý Streaming ETL trên AWS: So sánh chuyên sâu AWS Glue Streaming và Apache Spark Structured Streaming'
date: 2026-09-24 14:00:00 +0700
categories: [Data Engineering, Streaming]
tags: [AWS Glue, Apache Spark, Streaming ETL, Apache Kafka, AWS Kinesis, Data Engineering, PySpark]
keywords: [AWS Glue, Apache Spark, Streaming ETL, Apache Kafka, AWS Kinesis, PySpark]
pin: false
image:
  path: /assets/img/posts/2026/xu-ly-streaming-etl-tren-aws-glue-streaming-vs-spark-streaming/cover.webp
  alt: 'So sánh kiến trúc xử lý Streaming ETL giữa AWS Glue Streaming và Spark Structured Streaming'
---

# I. Dẫn nhập

Chào các bạn, trong thời đại dữ liệu vận hành theo thời gian thực (Real-time Enterprise), các đường ống xử lý theo lô truyền thống (Batch ETL) chạy mỗi đêm đã không còn đáp ứng đủ nhu cầu nghiệp vụ. Các hệ thống cảnh báo gian lận thẻ tín dụng (Fraud Detection), giám sát chỉ số sức khỏe của máy móc IoT, hay định giá vé máy bay động đều đòi hỏi dữ liệu sự kiện phải được thu thập, làm sạch và nạp vào Data Lakehouse chỉ trong vòng vài giây sau khi phát sinh.

Khi nhắc đến việc xử lý dòng sự kiện từ **Apache Kafka** hoặc **Amazon Kinesis Data Streams** trên nền tảng điện toán đám mây AWS, hai cái tên hàng đầu luôn được đưa lên bàn cân là:
1. **AWS Glue Streaming:** Giải pháp Serverless hoàn toàn do AWS đóng gói sẵn, được quảng cáo là "không cần quản trị cụm máy chủ, tự động scale theo tải, tích hợp liền mạch với Glue Data Catalog".
2. **Apache Spark Structured Streaming tự quản trị (Self-managed Spark):** Triển khai trực tiếp trên Amazon EKS (Kubernetes), Amazon EMR, hoặc cụm máy chủ ảo Amazon EC2.

Thế nhưng, trên chiến trường production thực tế, một nghịch lý kinh điển luôn lặp đi lặp lại:
- **Cú sốc hóa đơn với Glue Streaming:** Rất nhiều đội ngũ kỹ sư lựa chọn AWS Glue Streaming vì nghĩ rằng "Serverless thì nhàn nhã và tiết kiệm". Nhưng chỉ sau tháng đầu tiên vận hành, cả nhóm tá hỏa khi nhìn thấy hóa đơn AWS tăng vọt hàng ngàn USD! Lý do là vì Glue tính phí theo đơn vị DPU ($0.44 / DPU-giờ) với mức sàn tối thiểu là 2 DPUs cho mỗi job streaming. Vì streaming phải chạy liên tục 24/7 không ngừng nghỉ, **mức chi phí cứng cố định cho mỗi job đã lên tới ~$650 USD/tháng**, bất kể lưu lượng dữ liệu đổ về nhiều hay ít!
- **Cơn đau đầu vận hành với Spark tự quản:** Ngược lại, những team quyết định dựng Spark trên Kubernetes (EKS) để tận dụng máy chủ EC2 Spot giá rẻ thì lại chật vật với các bài toán vận hành hạ tầng cấp thấp (Day-2 Operations): căn chỉnh bộ nhớ JVM Executor, giải quyết lỗi Out of Memory (OOM), xử lý hiện tượng Checkpoint lag và tái cân bằng tải khi Kinesis Shard tăng đột biến.

Vậy giữa hai thái cực này, bản chất kỹ thuật ngầm của chúng khác nhau như thế nào? Cấu trúc `DynamicFrame` của Glue giải quyết bài toán Schema Drift thần thánh ra sao so với PySpark `DataFrame`? 

Trong bài viết này, mình sẽ cùng các bạn mổ xẻ tường tận kiến trúc ngầm, phân tích chi tiết ma trận trade-offs từ độ trễ, chi phí cho tới khả năng chịu lỗi, và cung cấp mã nguồn triển khai thực chiến cho cả hai nền tảng.

---

# II. Kiến trúc / Nguyên lý

Trước tiên, chúng ta cần làm sáng tỏ một sự thật kỹ thuật cơ bản: **AWS Glue Streaming thực chất chính là Apache Spark Structured Streaming được AWS bọc lại bên dưới một lớp vỏ trừu tượng hóa Serverless**.

```
+---------------------------------------------------------------------------------------------------+
|                        STREAMING ARCHITECTURE: AWS GLUE VS SELF-MANAGED SPARK                     |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|           +-------------------------------+       +-------------------------------+               |
|           |    Amazon Kinesis Data Stream |       |      Apache Kafka (MSK)       |               |
|           +---------------+---------------+       +---------------+---------------+               |
|                           |                                       |                               |
|                           +-------------------+-------------------+                               |
|                                               |                                                   |
|                                               v                                                   |
|                   +-------------------------------------------------------+                       |
|                   |           REAL-TIME EVENT INGESTION STREAM            |                       |
|                   +---------------------------+---------------------------+                       |
|                                               |                                                   |
|                       +-----------------------+-----------------------+                           |
|                       |                                               |                           |
|                       v                                               v                           |
|       +-------------------------------+               +-------------------------------+           |
|       |      AWS GLUE STREAMING       |               |    SPARK STRUCTURED STREAMING |           |
|       |          (Serverless)         |               |     (Self-managed on EKS/EMR) |           |
|       +-------------------------------+               +-------------------------------+           |
|       | * Managed Spark Micro-batch   |               | * Full Engine & Tuning Control|           |
|       | * DynamicFrame Schema Drift   |               | * Static Schema (Schema-on-w) |           |
|       | * Built-in Checkpointing      |               | * RocksDB State Store Provider|           |
|       | * Min 2 DPU ($650/mo floor)   |               | * Spot Instances (70% cheaper)|           |
|       | * 1 - 5s Micro-batch Latency  |               | * Sub-100ms or Continuous Mode|           |
|       +---------------+---------------+               +---------------+---------------+           |
|                       |                                               |                           |
|                       +-----------------------+-----------------------+                           |
|                                               |                                                   |
|                                               v                                                   |
|                               +-------------------------------+                                   |
|                               |      S3 DATA LAKEHOUSE        |                                   |
|                               | (Iceberg / Parquet / Delta)   |                                   |
|                               +-------------------------------+                                   |
+---------------------------------------------------------------------------------------------------+
```

### 1. Bản chất kỹ thuật của AWS Glue Streaming
Khi bạn khởi tạo một AWS Glue Streaming Job:
- AWS tự động cấp phát tài nguyên tính toán thông qua các đơn vị **DPU (Data Processing Unit)**. Một DPU tương đương với 4 vCPU và 16 GB bộ nhớ RAM.
- Worker Type thường là `G.1X` (1 DPU/worker) hoặc `G.2X` (2 DPUs/worker). Mức tối thiểu bắt buộc là 2 Workers (`number_of_workers = 2`), tương đương với 2 DPUs.
- Engine mặc định chạy theo cơ chế **Micro-batching**: Đọc từng mẻ dữ liệu nhỏ từ Kinesis Shards hoặc Kafka Topics theo chu kỳ thời gian (thường từ 1 đến 5 giây).

**Vũ khí tối thượng của Glue: `DynamicFrame` và `resolveChoice()`**
Trong PySpark tiêu chuẩn, `DataFrame` bắt buộc phải có một Schema định sẵn (Schema Enforcement). Nếu hệ thống nguồn vô tình gửi một bản ghi có kiểu dữ liệu sai lệch (ví dụ trường `user_id` vốn là số nguyên bỗng dưng gửi chuỗi `"N/A"`), PySpark Streaming job sẽ bị crash ngay lập tức hoặc ghi giá trị `NULL`.

AWS Glue giải quyết bài toán này bằng **`DynamicFrame`**:
- Mỗi bản ghi tự mang theo thông tin Schema của chính nó (Self-describing).
- Khi phát hiện một cột chứa nhiều kiểu dữ liệu hỗn tạp, `DynamicFrame` tự động tạo ra một kiểu dữ liệu đặc biệt gọi là **`choice` type** (chẳng hạn: `choice<int, string>`).
- Kỹ sư có thể sử dụng phương thức `resolveChoice()` để chỉ định quy tắc giải quyết xung đột mà không làm sập pipeline: ép kiểu về `cast:double`, chuyển vào trường phụ `project:user_id_str`, hoặc gom lại dưới dạng struct.

### 2. Bản chất kỹ thuật của Apache Spark Structured Streaming tự quản
Khi tự quản trị cụm Spark (chạy trên Kubernetes với Spark Operator hoặc Amazon EMR):
- **Toàn quyền can thiệp vào State Store:** Với các bài toán Stateful Streaming phức tạp (như tính toán Window Aggregation 1 giờ hoặc Stream-Stream Join), bộ nhớ RAM của Executor rất dễ bị cạn kiệt. Spark tự quản cho phép bạn chuyển đổi State Store Provider mặc định từ HDFS-backed sang **RocksDB State Store** lưu trữ trên ổ đĩa NVMe cục bộ, giúp duy trì hàng chục triệu trạng thái (states) mà không sợ OOM.
- **Hỗ trợ Continuous Processing Mode:** Ngoài Micro-batching, Spark Structured Streaming hỗ trợ chế độ xử lý dòng liên tục với độ trễ thấp ở mức dưới mili-giây (sub-millisecond latency).
- **Tối ưu hóa chi phí với Spot Instances:** Bạn có thể cấu hình Spark Driver chạy trên máy chủ On-Demand ổn định, còn toàn bộ Spark Executors chạy trên các máy ảo **EC2 Spot Instances** (tiết kiệm tới 70-80% chi phí điện toán).

### 3. Ma trận so sánh toàn diện

| Tiêu chí | AWS Glue Streaming | Apache Spark Streaming (EKS/EMR) |
| :--- | :--- | :--- |
| **Mô hình tính phí** | DPU ($0.44 / DPU-giờ). Sàn ~$650/tháng/job | EC2 / Spot Instances ($0.04 - $0.15/giờ) |
| **Độ phức tạp vận hành** | Gần như bằng 0 (Fully Serverless) | Cao (Tuning JVM, OS patch, Pod autoscaling) |
| **Độ trễ xử lý (Latency)**| 1 – 5 giây (Micro-batch) | < 100 mili-giây hoặc Continuous mode |
| **Xử lý Schema Drift** | Cực mạnh với `DynamicFrame` | Cần tự viết logic validate JSON schema |
| **Kiểm thử cục bộ (Local)**| Khó khăn (Phụ thuộc Glue Libraries) | Cực dễ (Chạy Docker, MinIO, Kafka local) |
| **Quản lý State Store** | Hạn chế cấu hình sâu | Hỗ trợ RocksDB, tối ưu cho State lớn |

---

# III. Cài đặt / Hands-on code

Bây giờ, mình sẽ cùng các bạn triển khai song song hai kịch bản: một bên là kịch bản AWS Glue Streaming xử lý dữ liệu Kinesis với DynamicFrame, bên kia là kịch bản PySpark Structured Streaming đọc từ Apache Kafka với Watermarking và Checkpointing.

### 1. Kịch bản AWS Glue Streaming Job (`glue_streaming_job.py`)

Kịch bản này chạy trên AWS Glue 4.0, đọc dữ liệu giao dịch từ Kinesis Data Stream, xử lý schema drift bằng `resolveChoice` và ghi ra S3 Data Lake:

```python
"""
AWS Glue Streaming ETL Job
Nhiệm vụ: Đọc từ Amazon Kinesis, xử lý Schema Drift bằng DynamicFrame và ghi ra S3
"""
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql import functions as F

# Khởi tạo ngữ cảnh Glue
args = getResolvedOptions(sys.argv, ["JOB_NAME"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

# Cấu hình nguồn Amazon Kinesis
kinesis_options = {
    "typeOfData": "kinesis",
    "streamARN": "arn:aws:kinesis:ap-southeast-1:123456789012:stream/payment-events",
    "classification": "json",
    "startingPosition": "LATEST",
    "inferSchema": "true"
}

print("[*] Đang khởi tạo kết nối Kinesis Data Stream...")
# Đọc streaming dưới dạng DynamicFrame
streaming_dynamic_frame = glueContext.create_data_frame.from_options(
    connection_type="kinesis",
    connection_options=kinesis_options,
    transformation_ctx="kinesis_source"
)

def process_batch(data_frame, batch_id):
    """
    Xử lý từng micro-batch dữ liệu
    """
    if data_frame.count() == 0:
        print(f"[*] Micro-batch {batch_id} rỗng, bỏ qua.")
        return

    print(f"[*] Đang xử lý Micro-batch {batch_id} với {data_frame.count()} bản ghi...")
    
    # Chuyển đổi sang DynamicFrame để sử dụng các phép biến đổi đặc quyền của Glue
    dynamic_frame = glueContext.create_dynamic_frame.from_rdd(
        data_frame.rdd, "batch_rdd"
    )
    
    # Xử lý Schema Drift: Nếu 'transaction_amount' bị lẫn chuỗi String và Double,
    # tự động ép kiểu toàn bộ về Double an toàn
    resolved_df = dynamic_frame.resolveChoice(
        specs=[("transaction_amount", "cast:double")],
        transformation_ctx="resolve_amount_choice"
    )
    
    # Chuyển ngược lại PySpark DataFrame để thêm các cột phân vùng
    clean_spark_df = resolved_df.toDF()
    clean_spark_df = clean_spark_df.withColumn(
        "processing_time", F.current_timestamp()
    ).withColumn(
        "year", F.year("processing_time")
    ).withColumn(
        "month", F.month("processing_time")
    )
    
    # Ghi dữ liệu ra S3 Data Lake dưới định dạng Parquet
    output_s3_path = "s3://my-lakehouse-bucket/silver/payment_events/"
    clean_spark_df.write \
        .mode("append") \
        .partitionBy("year", "month") \
        .parquet(output_s3_path)
        
    print(f"[✓] Đã ghi thành công Micro-batch {batch_id} ra S3!")

# Đăng ký xử lý micro-batch với chu kỳ cửa sổ 10 giây
query = streaming_dynamic_frame.writeStream \
    .format("console") \
    .foreachBatch(process_batch) \
    .trigger(processingTime="10 seconds") \
    .option("checkpointLocation", "s3://my-lakehouse-bucket/checkpoints/glue_payment_stream/") \
    .start()

query.awaitTermination()
job.commit()
```

### 2. Kịch bản PySpark Structured Streaming với Apache Kafka (`pyspark_streaming_kafka.py`)

Kịch bản này đại diện cho cụm Spark tự quản, đọc trực tiếp từ Apache Kafka (hoặc Amazon MSK), sử dụng Watermark để xử lý dữ liệu đến trễ và cấu hình Checkpoint:

```python
"""
Self-managed Apache Spark Structured Streaming Job
Nhiệm vụ: Đọc từ Apache Kafka, xử lý Watermarking và ghi stream ra S3 Parquet
"""
from pyspark.sql import SparkSession
from pyspark.sql.types import (
    StructType, StructField, StringType, 
    DoubleType, TimestampType
)
from pyspark.sql import functions as F

def create_spark_streaming_session():
    return (
        SparkSession.builder
        .appName("Spark-Kafka-Streaming-ETL")
        .config("spark.sql.streaming.stateStore.providerClass", 
                "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider")
        .config("spark.sql.shuffle.partitions", "20")
        .getOrCreate()
    )

def main():
    spark = create_spark_streaming_session()
    spark.sparkContext.setLogLevel("WARN")

    # 1. Định nghĩa Schema nghiêm ngặt cho sự kiện thanh toán
    payment_schema = StructType([
        StructField("transaction_id", StringType(), False),
        StructField("customer_id", StringType(), False),
        StructField("amount", DoubleType(), True),
        StructField("currency", StringType(), True),
        StructField("event_time", TimestampType(), True),
    ])

    print("[*] Đang kết nối tới cụm Apache Kafka...")
    
    # 2. Đọc luồng stream từ Kafka
    kafka_stream = (
        spark.readStream
        .format("kafka")
        .option("kafka.bootstrap.servers", "b-1.msk-cluster.amazonaws.com:9092")
        .option("subscribe", "online-payments")
        .option("startingOffsets", "latest")
        .option("failOnDataLoss", "false")
        .load()
    )

    # 3. Parse JSON payload và áp dụng Watermarking 15 phút
    parsed_stream = (
        kafka_stream
        .selectExpr("CAST(value AS STRING) as json_payload")
        .select(F.from_json(F.col("json_payload"), payment_schema).alias("data"))
        .select("data.*")
        .withWatermark("event_time", "15 minutes") # Cho phép dữ liệu trễ tối đa 15 phút
    )

    # 4. Thêm các cột phân vùng
    transformed_stream = (
        parsed_stream
        .filter(F.col("amount").isNotNull() & (F.col("amount") > 0))
        .withColumn("year", F.year("event_time"))
        .withColumn("month", F.month("event_time"))
    )

    # 5. Ghi stream ra S3 Data Lake với Trigger 30 giây
    s3_sink_path = "s3://my-lakehouse-bucket/silver/kafka_payments/"
    checkpoint_path = "s3://my-lakehouse-bucket/checkpoints/spark_kafka_payments/"

    print(f"[*] Bắt đầu streaming ra S3: {s3_sink_path}")
    streaming_query = (
        transformed_stream.writeStream
        .format("parquet")
        .outputMode("append")
        .partitionBy("year", "month")
        .option("path", s3_sink_path)
        .option("checkpointLocation", checkpoint_path)
        .trigger(processingTime="30 seconds")
        .start()
    )

    streaming_query.awaitTermination()

if __name__ == "__main__":
    main()
```

### 3. Cấu hình Terraform cho AWS Glue Streaming Job

Đoạn mã Terraform chuẩn hóa việc triển khai hạ tầng cho AWS Glue Streaming:

```hcl
resource "aws_glue_job" "payment_streaming_job" {
  name              = "payment-events-streaming-etl"
  role_arn          = "arn:aws:iam::123456789012:role/GlueStreamingExecutionRole"
  glue_version      = "4.0"
  worker_type       = "G.1X"
  number_of_workers = 2 # Mức sàn tối thiểu 2 DPUs = ~$650/tháng
  timeout           = 2880 # 48 giờ hoặc chạy vô tận

  command {
    name            = "gluestreaming"
    script_location = "s3://my-lakehouse-bucket/scripts/glue_streaming_job.py"
    python_version  = "3"
  }

  default_arguments = {
    "--job-language"                    = "python"
    "--continuous-log-logGroup"          = "/aws-glue/streaming-jobs"
    "--enable-continuous-cloudwatch-log" = "true"
    "--enable-metrics"                  = "true"
    "--TempDir"                         = "s3://my-lakehouse-bucket/temp/"
  }
}
```

---

# IV. Lesson learned / Tổng kết

Sau khi vận hành cả hai giải pháp trên quy mô hàng tỷ sự kiện mỗi ngày, mình đúc kết được 5 bài học sống còn:

1. **Tuyệt đối không bật Glue Streaming cho các luồng dữ liệu "thưa thớt":** Nếu luồng dữ liệu của bạn chỉ có vài sự kiện mỗi phút hoặc vài chục tin nhắn mỗi giờ, Glue Streaming sẽ thiêu đốt ngân sách của bạn một cách vô ích (~$650/tháng cho mỗi job). Trong trường hợp lưu lượng thấp hoặc không liên tục, hãy sử dụng **Amazon Kinesis Data Firehose** (chỉ tính phí theo dung lượng GB nạp thực tế) hoặc **AWS Lambda kết hợp SQS** để tiết kiệm tới 95% chi phí!
2. **Chọn Glue Streaming khi ưu tiên tốc độ ra mắt và Schema bất định:** Nếu nhóm của bạn ít người, không có chuyên gia DevOps/K8s chuyên biệt, và các nguồn dữ liệu bên thứ ba liên tục thay đổi cấu trúc không báo trước, Glue Streaming với tính năng `DynamicFrame.resolveChoice()` sẽ giúp bạn kê cao gối ngủ ngon mà không sợ nhận cuộc gọi lúc 3 giờ sáng vì pipeline bị crash.
3. **Chuyển sang Spark trên EKS khi quy mô từ 5 streaming jobs trở lên:** Khi doanh nghiệp mở rộng quy mô với hàng chục luồng dữ liệu streaming, việc duy trì từng Glue job riêng lẻ sẽ khiến chi phí phình to hàng chục ngàn USD. Gom toàn bộ các streaming jobs về một cụm **Amazon EKS chạy EC2 Spot Instances** sẽ giúp công ty tiết kiệm từ 70% đến 85% chi phí hàng tháng.
4. **Kiểm soát Processing Trigger để cứu vãn S3 API Costs:** Đừng bao giờ để trigger interval ở mức 1-2 giây nếu không thực sự có nhu cầu về độ trễ cực thấp. Mỗi lần một micro-batch ghi một file Parquet lên S3 sẽ tốn một request `PUT`. Nếu bạn chạy chu kỳ 1 giây, một ngày bạn sẽ gửi tới 86,400 PUT requests cho mỗi partition! Đặt khoảng nghỉ `.trigger(processingTime="30 seconds")` hoặc `60 seconds` vừa giúp gom các file Parquet to và nén tốt hơn, vừa giảm hóa đơn S3 API calls tới 30 lần.
5. **Giám sát sát sao độ trễ qua CloudWatch Metrics:** Hãy luôn thiết lập CloudWatch Alarms cho metric `glue.driver.streaming.batchProcessingTimeInMs` so với `streaming.batchTimeInMs`. Nếu thời gian xử lý một micro-batch bắt đầu vượt quá thời gian chu kỳ cửa sổ, đó là dấu hiệu của hiện tượng nghẽn cổ chai (Backpressure). Bạn cần tăng số lượng Kinesis Shards hoặc cấp phát thêm Worker DPU ngay lập tức để tránh làm trễ luồng dữ liệu.

Hy vọng bài so sánh chuyên sâu này đã giúp các bạn có được bức tranh rõ ràng và các căn cứ kỹ thuật vững chắc để lựa chọn giải pháp Streaming ETL phù hợp nhất cho tổ chức của mình!
