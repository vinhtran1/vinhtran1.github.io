---
title: 'Apache Spark Serverless trên AWS Lambda vs AWS EMR: Đánh đổi hiệu năng, giới hạn kiến trúc và bài toán chi phí'
date: 2026-09-24 11:00:00 +0700
categories: [Distributed Systems, Cloud Architecture]
tags: [Apache Spark, AWS Lambda, Amazon EMR, Serverless, Big Data, Cloud Cost Optimization]
keywords: [Apache Spark, AWS Lambda, Amazon EMR, Serverless, Cloud Cost Optimization]
pin: false
image:
  path: /assets/img/posts/2026/apache-spark-serverless-tren-aws-lambda-vs-aws-emr-danh-doi-va-chi-phi/cover.webp
  alt: 'So sánh Spark Serverless trên AWS Lambda và Amazon EMR Serverless về kiến trúc và bài toán chi phí'
---

# I. Dẫn nhập: Đặt vấn đề và Thách thức kỹ thuật

Chào các bạn, trong kỷ nguyên điện toán đám mây hiện đại, giấc mơ lớn nhất của mọi Data Engineer và Solutions Architect chính là: **"Zero-Infrastructure Big Data"**. Chúng ta khao khát một thế giới nơi không còn phải đau đầu tính toán kích thước cụm máy chủ (Cluster Sizing), không phải canh cánh nỗi lo vá lỗi bảo mật hệ điều hành (OS patching), không phải trả tiền cho hàng chục máy chủ EC2 nhàn rỗi lúc nửa đêm (Idle Server Cost), và tài nguyên điện toán tự động co giãn từ 0 lên hàng ngàn nhân vCPU rồi tự thu hồi ngay khi xử lý xong dữ liệu.

Nhắc đến xử lý dữ liệu lớn, **Apache Spark** chắc chắn là vị vua không thể bàn cãi. Tuy nhiên, việc vận hành một cụm Spark truyền thống (dù là trên EC2 tự dựng, Spark Standalone, hay Hadoop YARN) luôn là một gánh nặng vận hành khủng khiếp. Để khởi động một cụm Spark mới từ con số 0 thường mất từ 5 đến 15 phút. Điều này hoàn toàn đi ngược lại tính chất linh hoạt của các tác vụ vi mô (micro-batching) hoặc các luồng sự kiện (event-driven ETL) phát sinh rải rác trong ngày.

Từ đây, cộng đồng kỹ thuật hình thành hai trường phái tiếp cận mô hình Serverless cho Apache Spark trên nền tảng AWS:

```
+-----------------------------------------------------------------------------+
|                      HAI TRƯỜNG PHÁI SPARK SERVERLESS                       |
|                                                                             |
|  1. APACHE SPARK TRÊN AWS LAMBDA:           2. AMAZON EMR SERVERLESS:       |
|     - Đóng gói Docker Container (< 10GB)       - Managed Big Data Cluster   |
|     - Chạy mode local[*]                       - Phân tán Driver & Executor |
|     - Tối đa 10GB RAM, 6 vCPU, 15 phút         - Không giới hạn thời gian   |
|     - Cold start JVM: 15s - 30s!               - Tối ưu Distributed Shuffle |
|                                                                             |
|  -> CÂU HỎI THỰC CHIẾN: Đâu là Tipping Point hòa vốn chi phí & hiệu năng?  |
+-----------------------------------------------------------------------------+
```

1. **Trường phái "Ép voi vào lồng": Chạy PySpark trên AWS Lambda**: Tận dụng tính năng Lambda Container Image (hỗ trợ image lên tới 10GB, 10GB RAM, 6 vCPUs) để đóng gói JVM, Spark runtime và PySpark script vào một function serverless.
2. **Trường phái "Chính thống": Amazon EMR Serverless**: Dịch vụ chuyên dụng được AWS ra mắt nhằm giải phóng kỹ sư khỏi việc quản trị cụm máy chủ, tự động cấp phát và co giãn Driver và Executor theo đồ thị tính toán Directed Acyclic Graph (DAG) của Spark.

Thế nhưng, trong thực tế sản xuất, rất nhiều team kỹ thuật đã rơi vào những cái bẫy chết người vì thiếu hiểu biết tường tận:
- Có những team nghe theo các bài viết quảng cáo chạy Spark trên Lambda để "tiết kiệm chi phí", nhưng rồi ngã ngửa khi biết Lambda chỉ chạy được ở chế độ **Cluster 1 Node (`local[*]`)** — hoàn toàn triệt tiêu khả năng phân tán dữ liệu mạng!
- Có những team lại chọn EMR Serverless cho các batch dữ liệu chỉ vài chục Megabytes, để rồi lãng phí thời gian khởi động ứng dụng và chi phí tài nguyên tối thiểu không đáng có.

Trong bài viết này, mình sẽ cùng các bạn mổ xẻ bản chất cơ chế hoạt động của cả hai giải pháp, dựng mô hình toán học tìm ra **Tipping Point** (Điểm hòa vốn chi phí), và đặt ra một câu hỏi mang tính cách mạng: Tại sao chúng ta không dùng **DuckDB hoặc Polars trên Lambda** thay vì cố gắng ép một con voi Spark vào một chiếc lồng Serverless?

---

# II. Kiến trúc / Nguyên lý cốt lõi

### 1. Bản chất kỹ thuật của Spark trên AWS Lambda: "Con voi trong chiếc lồng"

Để hiểu rõ tại sao chạy Spark trên Lambda lại có những hạn chế nghiêm trọng, chúng ta cần nhìn thẳng vào cấu trúc thực thi của AWS Lambda:

```
 [AWS Lambda Container Execution Environment]
 +---------------------------------------------------------------+
 | vCPU: tối đa 6 Cores | RAM: tối đa 10,240 MB | /tmp: 10GB Max |
 | Thời gian sống tối đa: 15 Phút                                |
 |                                                               |
 | +-----------------------------------------------------------+ |
 | | JVM Process (Java JRE 17 Headless)                        | |
 | |  - Spark Driver (SparkContext, DAGScheduler)              | |
 | |  - Spark Executor (Chạy threads cục bộ: local[*])        | |
 | |  - Py4J Gateway Bridge                                    | |
 | +-----------------------------------------------------------+ |
 |            ^                                                  |
 |            | (IPC Pipe)                                       |
 | +-----------------------------------------------------------+ |
 | | Python 3.11 Runtime (Lambda Handler)                      | |
 | |  - PySpark DataFrame API                                  | |
 | |  - Boto3 / Data Ingest                                    | |
 | +-----------------------------------------------------------+ |
 +---------------------------------------------------------------+
         |
         x (KHÔNG CÓ KHẢ NĂNG GIAO TIẾP MẠNG GIỮA CÁC INSTANCES)
         |
 [Lambda Instance B]  <-- Không thể trao đổi Shuffle Block! -->  [Lambda Instance C]
```

Những rào cản vật lý không thể vượt qua (The Hard Ceilings):
- **Triệt tiêu hoàn toàn Shuffle Networking**: Trong kiến trúc Spark phân tán, các Executor trên các máy chủ khác nhau trao đổi các phân vùng dữ liệu qua mạng TCP trong các phép toán Shuffle (`groupByKey`, `reduceByKey`, `join`, `repartition`). Tuy nhiên, các Lambda instances là các sandbox cô lập, không có cơ chế giao tiếp trực tiếp ngang hàng (P2P networking). Do đó, Spark trên Lambda **chỉ có thể chạy ở chế độ `local[*]`**! Nó không phải là Spark phân tán, mà thực chất chỉ là một tiến trình đa luồng (multi-threaded process) trên một máy ảo duy nhất.
- **Giới hạn trần tài nguyên**:
  - RAM tối đa: $10,240 \text{ MB} = 10 \text{ GB}$. Sau khi trừ đi bộ nhớ hệ điều hành, JVM heap overhead, phần RAM khả dụng cho Spark chỉ còn khoảng 7GB - 8GB.
  - Dung lượng ổ đĩa đệm `/tmp`: Tối đa 10GB. Nếu dữ liệu trung gian bị spill ra đĩa vượt quá 10GB, Lambda sẽ lập tức sập với lỗi `No space left on device`.
  - Giới hạn thời gian (Timeout): Tuyệt đối **15 phút**. Bất kỳ job nào chạy đến phút thứ 15 đều bị AWS ngắt kết nối cưỡng bức.
- **Cơn ác mộng Cold Start**: Để chạy được PySpark, container phải nạp hệ điều hành Linux, tải máy ảo Java Virtual Machine (JVM), nạp hàng trăm JAR files của Spark, khởi tạo cầu nối Py4J Gateway, và thiết lập `SparkContext`. Quá trình này mất từ **15 đến 30 giây** trước khi dòng code xử lý đầu tiên của các bạn được chạy!

### 2. Bản chất kỹ thuật của Amazon EMR Serverless: Hệ thống phân tán thực thụ

Ngược lại hoàn toàn với Lambda, **Amazon EMR Serverless** là một dịch vụ PaaS phân tán hoàn chỉnh được AWS tối ưu riêng cho Apache Spark và Apache Hive:

```
 [Amazon EMR Serverless Application]
 +---------------------------------------------------------------+
 | Quản trị tự động: Không EC2, Tự động co giãn theo Stage DAG   |
 |                                                               |
 |  [Spark Driver Container]                                     |
 |  (Ví dụ: 2 vCPU, 4GB RAM)                                     |
 |           |                                                   |
 |           +------------------+------------------+             |
 |           | (Phân bổ Task)   | (Phân bổ Task)   |             |
 |           v                  v                  v             |
 |   [Spark Executor 1]  [Spark Executor 2]  [Spark Executor N]  |
 |   (4 vCPU, 16GB RAM)  (4 vCPU, 16GB RAM)  (Tối đa 1,000 vCPU) |
 |           |                  |                  |             |
 |           +------------------+------------------+             |
 |                              |                                |
 |           (Shuffle Blocks trao đổi qua mạng & S3)             |
 +---------------------------------------------------------------+
                                |
                   [Amazon S3 Cloud Data Lake]
```

Ưu thế vượt trội của EMR Serverless:
- **Khả năng Shuffle quy mô Petabyte**: Phân tách rõ ràng giữa Driver và nhiều Executor độc lập. Dữ liệu shuffle được lưu trữ và tối ưu trên bộ nhớ đệm tốc độ cao và Amazon S3.
- **Không giới hạn 15 phút**: Các job ETL nặng, tính toán mô hình Machine Learning có thể chạy liên tục hàng giờ liền mà không sợ bị timeout.
- **Pre-initialized Capacity**: Cho phép người dùng giữ ấm sẵn (warm pool) một lượng Worker nhất định. Khi có job gửi đến, Spark khởi động ngay lập tức trong vòng dưới **5 giây**, loại bỏ hoàn toàn nhược điểm cold start.

### 3. Mô hình toán học phân tích chi phí và Tipping Point

Để quyết định lựa chọn giải pháp nào, chúng ta không thể dựa vào cảm tính mà phải dùng toán học chi phí chi tiết của AWS (tính theo đơn giá tại khu vực `us-east-1`):

#### Công thức chi phí AWS Lambda (Architecture x86_64, RAM 10GB, vCPU ~6):
Đơn giá bộ nhớ: $\$0.0000166667 \text{ / GB-giây}$.
Với cấu hình 10GB RAM:
$$\text{Cost}_{\lambda}(\text{duration}) = \text{duration (giây)} \times 10 \times 0.0000166667 + 0.0000002 \text{ (request fee)}$$
$$\text{Cost}_{\lambda}(\text{duration}) \approx \text{duration} \times \$0.00016667 \text{ / giây} \approx \mathbf{\$0.60 \text{ / giờ}}$$

#### Công thức chi phí Amazon EMR Serverless:
- vCPU-hour: $\$0.052624$ ($\approx \$0.00001462 \text{ / vCPU-giây}$)
- Memory GB-hour: $\$0.0057785$ ($\approx \$0.00000161 \text{ / GB-giây}$)

Giả sử một job EMR Serverless chạy với cấu hình tối thiểu tương đương Lambda (Driver: 2 vCPU + 4GB RAM, 1 Executor: 4 vCPU + 8GB RAM $\rightarrow$ Tổng cộng: 6 vCPU + 12GB RAM):
$$\text{Cost}_{EMR}(\text{duration}) = \text{duration} \times (6 \times 0.00001462 + 12 \times 0.00000161) \approx \text{duration} \times \mathbf{\$0.000107 \text{ / giây}} \approx \mathbf{\$0.385 \text{ / giờ}}$$

```
 CHI PHÍ THEO THỜI GIAN THỰC THI (USD)
 ^
 |                               / Spark on Lambda ($0.60/h + Cold Start Tax)
 |                              / 
 |                             /  <--- TIPPING POINT (~3 - 5 phút)
 |                            /
 |     ....................../........................ EMR Serverless ($0.385/h)
 |    /                     /
 |   / (EMR App Init)      /
 |  /                     /
 | /                     /  DuckDB on Lambda ($0.015/h, Khởi động 30ms)
 |/_____________________/___________________________> Thời gian chạy
```

**Phân tích điểm hòa vốn (The Tipping Point)**:
- **Dưới 2 phút thực thi (Micro-jobs)**: Lambda có lợi thế về chi phí nếu đã được giữ ấm, nhưng nếu tính cả thời gian 25 giây cold start của JVM, người dùng đang phải trả tiền oan cho 25 giây vô ích của 10GB RAM!
- **Từ 3 phút đến 15 phút**: Đơn giá tính toán của EMR Serverless cho cùng một lượng vCPU/RAM rẻ hơn Lambda tới **36%** ($\$0.385/\text{h}$ so với $\$0.60/\text{h}$).
- **Trên 15 phút**: Lambda hoàn toàn bị loại khỏi cuộc chơi do chạm giới hạn cứng của nền tảng.

### 4. Kẻ soán ngôi bất ngờ: DuckDB và Polars trên AWS Lambda

Nếu các bạn chỉ cần xử lý các file dữ liệu dưới 5GB trên một node đơn lẻ bên trong Lambda, tại sao lại phải chịu đựng một container nặng 2GB chứa JVM, Hadoop và PySpark?

Hãy so sánh với **DuckDB** hoặc **Polars**:
- Được viết bằng C++ và Rust nguyên bản, không cần máy ảo Java (No JVM).
- Dung lượng thư viện Python chỉ khoảng **30MB** (thay vì 1.5GB của Spark).
- **Cold start chỉ mất 30 đến 50 mili-giây** (nhanh gấp 500 lần Spark!).
- Tiêu tốn chỉ từ 512MB đến 2GB RAM cho các tác vụ tổng hợp dữ liệu Parquet lớn nhờ kỹ thuật Vectorized Execution và bộ quản lý bộ nhớ đệm Out-of-Core.

---

# III. Cài đặt / Hands-on code & Tối ưu thực chiến

Dưới đây là toàn bộ mã nguồn thực tế để triển khai và đo lường cả hai giải pháp.

### 1. Multi-Stage Dockerfile tối ưu cho PySpark trên AWS Lambda

Để nén nhỏ kích thước Docker image của Spark và tối ưu thời gian khởi động, chúng ta sử dụng kỹ thuật multi-stage build kết hợp với Java Headless:

```dockerfile
# Stage 1: Runtime base với Python 3.11 và Java 17 headless
FROM public.ecr.aws/lambda/python:3.11 AS builder

# Cài đặt OpenJDK 17 headless tinh gọn
RUN yum install -y java-17-amazon-corretto-headless tar gzip && \
    yum clean all

# Thiết lập biến môi trường Java & Spark
ENV JAVA_HOME="/usr/lib/jvm/java-17-amazon-corretto"
ENV SPARK_VERSION="3.5.1"
ENV HADOOP_VERSION="3"
ENV SPARK_HOME="/opt/spark"
ENV PATH="${PATH}:${JAVA_HOME}/bin:${SPARK_HOME}/bin:${SPARK_HOME}/sbin"

# Tải và lược bỏ các thành phần không cần thiết của Spark (tài liệu, examples, R library)
RUN mkdir -p ${SPARK_HOME} && \
    curl -sL "https://archive.apache.org/dist/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz" | \
    tar -xz --strip-components=1 -C ${SPARK_HOME} && \
    rm -rf ${SPARK_HOME}/examples ${SPARK_HOME}/docs ${SPARK_HOME}/R

# Stage 2: Final Production Container Image
FROM public.ecr.aws/lambda/python:3.11

COPY --from=builder /usr/lib/jvm/java-17-amazon-corretto /usr/lib/jvm/java-17-amazon-corretto
COPY --from=builder /opt/spark /opt/spark

ENV JAVA_HOME="/usr/lib/jvm/java-17-amazon-corretto"
ENV SPARK_HOME="/opt/spark"
ENV PYTHONPATH="${SPARK_HOME}/python:${SPARK_HOME}/python/lib/py4j-0.10.9.7-src.zip:${PYTHONPATH}"

# Cài đặt thư viện Python tối thiểu
RUN pip install --no-cache-dir pyspark==3.5.1 boto3 pyarrow

# Copy mã nguồn xử lý vào Lambda Task Root
COPY app.py ${LAMBDA_TASK_ROOT}

CMD ["app.lambda_handler"]
```

### 2. Kịch bản Lambda Handler: Thực thi PySpark Local Mode an toàn

Trong file `app.py`, chúng ta khởi tạo SparkSession với các cờ tối ưu hóa bộ nhớ cục bộ, tránh rò rỉ RAM:

```python
import os
import time
import logging
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum as _sum, avg

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Tái sử dụng SparkSession giữa các đợt warm invocations
spark = None

def get_spark_session():
    global spark
    if spark is None:
        start_init = time.time()
        spark = SparkSession.builder \
            .master("local[*]") \
            .appName("Lambda-PySpark-Job") \
            .config("spark.driver.memory", "7g") \
            .config("spark.sql.shuffle.partitions", "4") \
            .config("spark.ui.enabled", "false") \
            .config("spark.driver.extraJavaOptions", "-XX:+UseG1GC -XX:+ExitOnOutOfMemoryError") \
            .getOrCreate()
        logger.info(f"SparkSession initialized in {time.time() - start_init:.2f} seconds.")
    return spark

def lambda_handler(event, context):
    start_time = time.time()
    s3_input_uri = event.get("input_uri")
    s3_output_uri = event.get("output_uri")

    if not s3_input_uri or not s3_output_uri:
        return {"statusCode": 400, "body": "Missing input_uri or output_uri"}

    try:
        ss = get_spark_session()
        
        # Đọc dữ liệu từ S3
        df = ss.read.parquet(s3_input_uri)
        
        # Thực thi tổng hợp thống kê
        aggregated_df = df.filter(col("amount") > 0) \
            .groupBy("category") \
            .agg(
                _sum("amount").alias("total_sales"),
                avg("amount").alias("avg_sales")
            )
        
        # Ghi kết quả về S3
        aggregated_df.write.mode("overwrite").parquet(s3_output_uri)
        
        duration = time.time() - start_time
        logger.info(f"Execution finished in {duration:.2f}s")
        return {"statusCode": 200, "duration_seconds": duration}
    except Exception as e:
        logger.error(f"Spark job failed: {str(e)}", exc_info=True)
        return {"statusCode": 500, "error": str(e)}
```

### 3. Kịch bản Submit Job lên EMR Serverless bằng Boto3

Dưới đây là kịch bản Python tự động tạo EMR Serverless Application kiến trúc ARM64 (Graviton) và submit một Spark job:

```python
import boto3
import time
import os

emr_client = boto3.client("emr-serverless", region_name="us-east-1")

APPLICATION_NAME = "ad-tech-spark-serverless"
EXECUTION_ROLE_ARN = os.environ.get("EMR_EXECUTION_ROLE_ARN")
S3_SCRIPT_URI = "s3://my-bigdata-bucket/scripts/etl_aggregation.py"

def create_emr_application():
    """Tạo EMR Serverless Application sử dụng CPU Graviton2 tiết kiệm 20% chi phí."""
    response = emr_client.create_application(
        name=APPLICATION_NAME,
        releaseLabel="emr-7.1.0",
        type="SPARK",
        architecture="ARM64",
        autoStartConfiguration={"enabled": True},
        autoStopConfiguration={"enabled": True, "idleTimeoutMinutes": 5},
        initialCapacity={
            "DRIVER": {
                "workerCount": 1,
                "workerConfiguration": {"cpu": "2vCPU", "memory": "4GB"}
            },
            "EXECUTOR": {
                "workerCount": 2,
                "workerConfiguration": {"cpu": "4vCPU", "memory": "8GB"}
            }
        }
    )
    app_id = response["applicationId"]
    print(f"Created EMR Serverless App ID: {app_id}")
    return app_id

def submit_spark_job(app_id, input_path, output_path):
    response = emr_client.start_job_run(
        applicationId=app_id,
        executionRoleArn=EXECUTION_ROLE_ARN,
        jobDriver={
            "sparkSubmit": {
                "entryPoint": S3_SCRIPT_URI,
                "entryPointArguments": [input_path, output_path],
                "sparkSubmitParameters": (
                    "--conf spark.executor.cores=4 "
                    "--conf spark.executor.memory=8g "
                    "--conf spark.driver.cores=2 "
                    "--conf spark.driver.memory=4g "
                    "--conf spark.dynamicAllocation.enabled=true"
                )
            }
        },
        configurationOverrides={
            "monitoringConfiguration": {
                "s3MonitoringConfiguration": {
                    "logUri": "s3://my-bigdata-bucket/emr-logs/"
                }
            }
        }
    )
    job_run_id = response["jobRunId"]
    print(f"Submitted Job Run ID: {job_run_id}")
    return job_run_id
```

### 4. Mô phỏng So sánh Chi phí & Hiệu năng: Lambda vs EMR vs DuckDB

Dưới đây là đoạn script Python mô phỏng chính xác chi phí của 3 giải pháp theo dung lượng và thời gian thực thi:

```python
def calculate_costs(duration_sec, data_size_gb):
    # 1. AWS Lambda 10GB RAM (6 vCPU)
    lambda_rate_per_sec = 10 * 0.0000166667
    lambda_cost = (duration_sec * lambda_rate_per_sec) + 0.0000002

    # 2. EMR Serverless (Driver: 2vCPU+4GB, 2 Executors: 8vCPU+16GB => 10 vCPU, 20GB RAM)
    vcpu_rate = 0.052624 / 3600
    mem_rate = 0.0057785 / 3600
    emr_cost = duration_sec * (10 * vcpu_rate + 20 * mem_rate)

    # 3. DuckDB on Lambda (Chỉ cần 2GB RAM và thời gian chạy nhanh gấp 3 lần do không có JVM overhead)
    duckdb_duration = max(duration_sec / 3, 2.0)
    duckdb_cost = (duckdb_duration * (2 * 0.0000166667)) + 0.0000002

    return {
        "duration_sec": duration_sec,
        "lambda_spark_usd": round(lambda_cost, 6),
        "emr_serverless_usd": round(emr_cost, 6),
        "duckdb_lambda_usd": round(duckdb_cost, 6)
    }

if __name__ == "__main__":
    test_cases = [30, 60, 180, 300, 600, 900]
    print(f"{'Duration':<10} | {'Spark on Lambda':<18} | {'EMR Serverless':<18} | {'DuckDB on Lambda':<18}")
    print("-" * 72)
    for dur in test_cases:
        res = calculate_costs(dur, 2.0)
        print(f"{dur}s{'':<7} | ${res['lambda_spark_usd']:<17} | ${res['emr_serverless_usd']:<17} | ${res['duckdb_lambda_usd']:<17}")
```

---

# IV. Lesson learned / Tổng kết & Best Practices

Sau khi đồng hành cùng nhiều dự án chuyển đổi hạ tầng dữ liệu lên đám mây, mình xin đúc kết 5 quy tắc vàng giúp các bạn đưa ra lựa chọn sáng suốt:

### 1. Đừng dùng Spark trên Lambda nếu có thể chọn DuckDB hoặc Polars
Chạy Spark trên Lambda là một giải pháp tình thế đầy khiên cưỡng. Nếu logic tính toán của các bạn chỉ gói gọn trong các phép toán SQL tổng hợp, lọc dữ liệu, nối bảng (join) trên các tập dữ liệu dưới 5GB, hãy chuyển ngay sang **DuckDB** hoặc **Polars**. Bạn sẽ tiết kiệm được **90% chi phí**, triệt tiêu hoàn toàn thời gian cold start 25 giây, và giải phóng hệ thống khỏi sự cồng kềnh của JVM.

### 2. Chọn EMR Serverless khi kích thước dữ liệu có độ biến thiên cao (Workload Skew)
EMR Serverless là sự lựa chọn hoàn hảo cho các pipeline ETL theo lịch chạy định kỳ (chẳng hạn như chạy mỗi đêm một lần), nơi khối lượng dữ liệu biến động từ vài chục Gigabytes đến hàng chục Terabytes. Cơ chế tự động cấp phát Executor theo nhu cầu thực tế của từng Stage giúp các bạn không lãng phí dù chỉ một xu cho tài nguyên nhàn rỗi.

### 3. Tận dụng kiến trúc ARM64 Graviton ở mọi nơi có thể
Cả AWS Lambda lẫn Amazon EMR Serverless đều hỗ trợ vi xử lý AWS Graviton dựa trên kiến trúc ARM64. Chỉ bằng việc bật cờ `architecture="ARM64"`, các bạn ngay lập tức nhận được:
- Hiệu năng xử lý tính toán tăng từ 15% đến 25%.
- Đơn giá trên mỗi giây điện toán rẻ hơn **20%** so với x86 truyền thống.

### 4. Khai thác tính năng Pre-initialized Capacity một cách thông minh
Nếu bắt buộc phải chọn EMR Serverless cho các pipeline yêu cầu độ trễ thấp (SLA dưới 1 phút), hãy cấu hình `initialCapacity` trong các khung giờ cao điểm (Peak Hours). Đồng thời, hãy nhớ thiết lập tính năng tự động tắt máy (`idleTimeoutMinutes: 5`) để cụm máy chủ tự động thu hồi ngay khi hết giờ cao điểm.

### 5. Bảng ma trận quyết định nhanh (Decision Matrix)

| Tiêu chí | DuckDB / Polars trên Lambda | PySpark trên AWS Lambda | Amazon EMR Serverless |
| :--- | :--- | :--- | :--- |
| **Kích thước dữ liệu** | $< 5 \text{ GB}$ | $2 \text{ GB} - 10 \text{ GB}$ | $> 10 \text{ GB}$ (Lên tới hàng TB) |
| **Thời gian chạy** | $< 2 \text{ phút}$ | $2 - 10 \text{ phút}$ | Không giới hạn ($> 15 \text{ phút}$) |
| **Yêu cầu Shuffle** | Không cần | Không hỗ trợ | Bắt buộc (Phân tán nhiều nodes) |
| **Thời gian Cold Start** | Siêu nhanh ($30\text{ms}$) | Chậm ($15\text{s} - 30\text{s}$) | Nhanh ($< 5\text{s}$ nếu giữ ấm) |
| **Rào cản quản trị** | Không | Trung bình | Không |

Hy vọng bài so sánh chi tiết này sẽ giúp các bạn tự tin lựa chọn công cụ phù hợp nhất cho bài toán kiến trúc của mình, cân bằng hoàn hảo giữa hiệu năng kỹ thuật và bài toán tối ưu chi phí cho doanh nghiệp!
