---
title: 'Data Lakehouse Series - Bài 3: Chuẩn hóa dữ liệu thô vào Silver Layer với AWS Lambda và SQS Event-Driven'
date: 2026-09-24 12:00:00 +0700
categories: [Tutorial, Data Engineering]
tags: [Data Lakehouse, AWS, S3, SQS, AWS Lambda, Python, Event-Driven, Parquet]
keywords: [Data Lakehouse, AWS, S3, SQS, AWS Lambda, Parquet]
pin: false
image:
  path: /assets/img/posts/2026/data-lakehouse-series-bai-3-normalize-du-lieu-aws-lambda-sqs/cover.webp
  alt: 'Kiến trúc chuẩn hóa dữ liệu S3 Raw sang Silver qua SQS và AWS Lambda Controller'
---

# I. Dẫn nhập

Chào các bạn, rất vui được gặp lại các bạn trong bài viết số 3 của chuỗi bài thực chiến xây dựng **End-to-End Data Lakehouse trên hạ tầng AWS**.

Ở hai bài viết trước, chúng ta đã cùng nhau đặt những viên gạch nền móng đầu tiên cho nền tảng dữ liệu:
- **Bài 1:** Thiết kế kiến trúc tổng thể theo mô hình Medallion (Bronze/Raw -> Silver -> Gold), định hình mô hình dữ liệu Star Schema và phân chia các tầng lưu trữ trên Amazon S3.
- **Bài 2:** Triển khai cơ chế Ingestion thu nạp dữ liệu thô đa nguồn từ Company A (streaming qua Kafka) và Company B (batch logs) đổ dồn về S3 Raw Bucket tại đường dẫn `s3://data-lakehouse-pzoscg/raw/...`.

Hiện tại, kho dữ liệu thô (Bronze Layer) của chúng ta đã chứa hàng trăm nghìn file JSON và CSV đổ về liên tục. Nhưng câu hỏi đặt ra là: **Tại sao chúng ta không thể đưa thẳng dữ liệu thô này cho các chuyên viên phân tích (Data Analyst) hay mô hình Machine Learning sử dụng ngay?**

Thực tế dữ liệu thô luôn ẩn chứa những "cơn ác mộng":
1. **Định dạng hỗn loạn và Schema Drift:** Cùng một trường thông tin ngày tháng, nguồn Company A gửi định dạng `DD/MM/YYYY`, nguồn Company B gửi Unix Epoch milliseconds, thậm chí có những bản ghi bị lỗi chuỗi rỗng (`""`).
2. **Kiểu dữ liệu thiếu an toàn:** Tiền tệ bị lưu dưới dạng chuỗi String có ký hiệu tiền tệ (`"$1,250.50"`), các giá trị số thập phân bị làm tròn sai lệch.
3. **Vấn nạn Small Files:** Hàng triệu file JSON kích thước chỉ vài chục Kilobytes trên S3 sẽ khiến các công cụ tính toán phân tán như Apache Spark hay AWS Athena bị tê liệt vì chi phí overhead mở và đóng file quá lớn.

Chính vì vậy, nhiệm vụ cốt tử của **Bài 3** này là xây dựng tầng chuyển hóa **Silver Layer**: tự động làm sạch (Cleanse), chuẩn hóa kiểu dữ liệu, khử trùng lặp, và nén toàn bộ thành các file cột tối ưu **Apache Parquet (Snappy compressed)** có phân vùng vật lý.

Thay vì chạy các batch job định kỳ nặng nề, mình sẽ hướng dẫn các bạn thiết kế một kiến trúc **Event-Driven Serverless** cực kỳ tinh gọn và chi phí siêu rẻ: kết hợp **Amazon S3 Event Notifications**, **Amazon SQS Message Queue**, và mô hình **AWS Lambda Controller - Worker Pattern**.

---

# II. Kiến trúc / Nguyên lý

Trước khi bắt tay vào viết code, hãy cùng nhau phân tích luồng chuyển động của dữ liệu và lý do tại sao kiến trúc Event-Driven kết hợp với hàng đợi SQS lại là sự lựa chọn tối ưu nhất.

```
+---------------------------------------------------------------------------------------------------+
|                           EVENT-DRIVEN NORMALIZATION ARCHITECTURE                                 |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|    +-----------------------------+                                                                |
|    |        S3 RAW BUCKET        |                                                                |
|    |  (raw/companyA/customer/..) |                                                                |
|    +--------------+--------------+                                                                |
|                   |                                                                               |
|                   | 1. s3:ObjectCreated:*                                                         |
|                   v                                                                               |
|    +-----------------------------+         +-------------------------------+                      |
|    |    Amazon SQS (Buffer)      |-------->|    Dead Letter Queue (DLQ)    |                      |
|    |       [RawDataQueue]        | (Err>3) |      [RawDataQueue-DLQ]       |                      |
|    +--------------+--------------+         +-------------------------------+                      |
|                   |                                                                               |
|                   | 2. Event Source Mapping (Batch = 10 msgs)                                     |
|                   v                                                                               |
|    +-----------------------------+                                                                |
|    |   AWS Lambda: Controller    |                                                                |
|    |     (SQSMessageGetter)      |                                                                |
|    +--------------+--------------+                                                                |
|                   |                                                                               |
|                   | 3. Dynamic Routing & Async Invocation                                         |
|                   +-------------------------------+-------------------------------+               |
|                   |                               |                               |               |
|                   v                               v                               v               |
|    +-----------------------------+ +-----------------------------+ +----------------------------+ |
|    | Lambda Worker: Customer     | | Lambda Worker: Transactions | | Lambda Worker: CompanyB    | |
|    | (Cleanse & Parquet Schema)  | | (Cleanse & Parquet Schema)  | | (Cleanse & Parquet Schema) | |
|    +--------------+--------------+ +--------------+--------------+ +--------------+-------------+ |
|                   |                               |                               |               |
|                   +-------------------------------+-------------------------------+               |
|                                                   |                                               |
|                                                   | 4. Write Snappy Parquet                       |
|                                                   v                                               |
|                                    +-----------------------------+                                |
|                                    |       S3 SILVER BUCKET      |                                |
|                                    |   (silver/customer/year=../)|                                |
|                                    +-----------------------------+                                |
+---------------------------------------------------------------------------------------------------+
```

### 1. Tại sao phải có SQS kẹp giữa S3 và AWS Lambda?
Nhiều bạn thường thắc mắc: *"Tại sao không cấu hình S3 Event Notification kích hoạt trực tiếp AWS Lambda cho nhanh?"*

Đây là một lỗi kiến trúc rất phổ biến. Khi hệ thống của bạn hoạt động bình thường, việc gọi trực tiếp hoạt động ổn. Nhưng khi có sự kiện bất thường (ví dụ: đối tác đẩy dồn một lúc 50,000 files vào S3 Raw), 50,000 Lambda functions sẽ được gọi đồng thời, ngay lập tức vượt quá ngưỡng **Account Concurrency Limit** (mặc định 1,000 concurrent executions). Hậu quả là Lambda bị throttle và làm gián đoạn mọi dịch vụ khác trong cùng tài khoản AWS.

Việc đưa **Amazon SQS** vào giữa mang lại 3 lợi ích vô giá:
1. **Hấp thụ xung đột tải (Rate Limiting & Smoothing):** SQS hoạt động như một hồ chứa đệm, giữ lại toàn bộ sự kiện và cung cấp cho Lambda xử lý từ từ theo năng lực đã cấu hình (`ReservedConcurrency`).
2. **Cơ chế Dead Letter Queue (DLQ):** Nếu một file dữ liệu bị lỗi hỏng (corrupted JSON) khiến Lambda xử lý thất bại 3 lần liên tiếp, SQS sẽ tự động đưa message vào DLQ để kỹ sư điều tra sau, không làm nghẽn toàn bộ đường ống.
3. **Tiết kiệm chi phí nhờ Batching:** SQS cho phép cấu hình gom cụm (ví dụ: 10 messages/lần gọi Lambda). Thay vì tốn 10 lượt invoke Lambda, chúng ta chỉ tốn 1 lượt invoke để xử lý 10 files!

### 2. Mô hình Controller - Worker Pattern
Để đảm bảo nguyên tắc phân tách trách nhiệm (Single Responsibility Principle), chúng ta chia pipeline thành 2 tầng Lambda:
- **Lambda Controller (`SQSMessageGetter`):** Giữ vai trò nhạc trưởng điều phối. Controller chỉ chịu trách nhiệm đọc batch messages từ SQS, giải mã payload JSON của S3 Event, bóc tách S3 Bucket và Object Key. Dựa vào tiền tố prefix của đường dẫn (`raw/companyA/customer/` hay `raw/companyA/transactions/`), Controller sẽ kích hoạt không đồng bộ (`InvocationType='Event'`) đến đúng Worker tương ứng.
- **Lambda Worker (`normalize_worker`):** Chịu trách nhiệm thực thi các phép tính toán nặng (Compute-heavy). Worker tải file thô về bộ nhớ, làm sạch dữ liệu, ép kiểu nghiêm ngặt theo schema quy định, chuyển đổi thành Apache Parquet nén Snappy, và ghi ra S3 Silver Layer theo cấu trúc phân vùng chuẩn hóa `year=YYYY/month=MM/`.

### 3. Tiêu chuẩn dữ liệu tại Silver Layer
Dữ liệu khi cập bến S3 Silver phải tuân thủ nghiêm ngặt các quy chuẩn:
- **Định dạng file:** 100% là Apache Parquet, nén bằng thuật toán Snappy (đạt tỷ lệ cân bằng hoàn hảo giữa kích thước nén và tốc độ giải nén của CPU).
- **Chuẩn hóa thời gian:** Toàn bộ cột thời gian được chuyển về kiểu dữ liệu Timestamp chuẩn UTC theo định dạng ISO-8601.
- **Tính lũy đẳng (Idempotency):** Tên file Parquet ghi ra tại tầng Silver được gắn mã băm (MD5) của đường dẫn file Raw gốc. Nhờ vậy, nếu một file thô vô tình bị xử lý 2 lần, file mới tại tầng Silver sẽ ghi đè chính xác lên file cũ mà không làm trùng lặp số lượng bản ghi.

---

# III. Cài đặt / Hands-on code

Bây giờ, mình sẽ cùng các bạn triển khai toàn bộ giải pháp từ mã nguồn hạ tầng Terraform cho tới code xử lý Python của Lambda.

### 1. Khởi tạo Hạ tầng với Terraform

File cấu hình Terraform thiết lập SQS Queue, DLQ, quyền IAM và S3 Notification:

```hcl
# main.tf: Hạ tầng SQS, S3 Event và IAM Policy

# 1. Dead Letter Queue
resource "aws_sqs_queue" "raw_data_dlq" {
  name                      = "data-lakehouse-raw-dlq"
  message_retention_seconds = 1209600 # 14 ngày
}

# 2. Main SQS Queue với cấu hình Redrive Policy
resource "aws_sqs_queue" "raw_data_queue" {
  name                       = "data-lakehouse-raw-queue"
  visibility_timeout_seconds = 900 # 15 phút (gấp 6 lần timeout của Lambda Controller)
  message_retention_seconds  = 345600 # 4 ngày
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.raw_data_dlq.arn
    maxReceiveCount     = 3
  })
}

# Policy cho phép S3 gửi message vào SQS
resource "aws_sqs_queue_policy" "sqs_s3_policy" {
  queue_url = aws_sqs_queue.raw_data_queue.id
  policy    = jsonencode({
    Version   = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.raw_data_queue.arn
      Condition = {
        ArnEquals = {
          "aws:SourceArn" = "arn:aws:s3:::data-lakehouse-pzoscg"
        }
      }
    }]
  })
}

# 3. Cấu hình S3 Event Notification
resource "aws_s3_bucket_notification" "raw_bucket_notification" {
  bucket = "data-lakehouse-pzoscg"

  queue {
    queue_arn     = aws_sqs_queue.raw_data_queue.arn
    events        = ["s3:ObjectCreated:*"]
    filter_prefix = "raw/"
  }
}
```

### 2. Triển khai Lambda Controller (`lambda_controller.py`)

Hàm này được kích hoạt thông qua **SQS Event Source Mapping**, phân loại đường dẫn và gọi Worker tương ứng:

```python
"""
Lambda Controller: SQSMessageGetter
Nhiệm vụ: Đọc batch event từ SQS, định tuyến và kích hoạt Lambda Worker
"""
import json
import logging
import urllib.parse
import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

lambda_client = boto3.client("lambda")

# Mapping giữa tiền tố thư mục S3 và tên hàm Lambda Worker xử lý
ROUTING_MAP = {
    "raw/companyA/customer": "normalize_customer_worker",
    "raw/companyA/transactions": "normalize_transaction_worker",
    "raw/companyB/orders": "normalize_company_b_worker",
}

def lambda_handler(event, context):
    logger.info(f"Nhận được {len(event.get('Records', []))} bản ghi từ SQS")
    batch_item_failures = []

    for record in event.get("Records", []):
        message_id = record["messageId"]
        try:
            body = json.loads(record["body"])
            
            # S3 Event Notification bọc danh sách bản ghi trong key 'Records'
            for s3_record in body.get("Records", []):
                bucket_name = s3_record["s3"]["bucket"]["name"]
                # URL decode để xử lý các ký tự đặc biệt như khoảng trắng hoặc dấu tiếng Việt
                object_key = urllib.parse.unquote_plus(
                    s3_record["s3"]["object"]["key"]
                )
                
                logger.info(f"Đang phân tích đối tượng: s3://{bucket_name}/{object_key}")
                
                # Tìm kiếm Worker phù hợp dựa trên prefix
                target_worker = None
                for prefix, worker_func in ROUTING_MAP.items():
                    if object_key.startswith(prefix):
                        target_worker = worker_func
                        break
                
                if target_worker:
                    payload = {
                        "bucket": bucket_name,
                        "key": object_key,
                        "timestamp": s3_record["eventTime"],
                    }
                    # Kích hoạt bất đồng bộ (Fire-and-Forget)
                    lambda_client.invoke(
                        FunctionName=target_worker,
                        InvocationType="Event",
                        Payload=json.dumps(payload),
                    )
                    logger.info(f"Đã chuyển tiếp tới Worker: {target_worker}")
                else:
                    logger.warning(f"Bỏ qua: Không tìm thấy Worker cho prefix: {object_key}")
                    
        except Exception as exc:
            logger.error(f"Lỗi khi xử lý message {message_id}: {str(exc)}", exc_info=True)
            # Báo lỗi để SQS chỉ retry lại duy nhất message bị fail này
            batch_item_failures.append({"itemIdentifier": message_id})

    return {"batchItemFailures": batch_item_failures}
```

### 3. Triển khai Lambda Worker Chuẩn hóa (`normalize_customer_worker.py`)

Hàm này sử dụng thư viện **AWS SDK for pandas (`awswrangler`)** để đọc dữ liệu thô, thực hiện chuẩn hóa kiểu dữ liệu và ghi ra file Parquet có phân vùng:

```python
"""
Lambda Worker: normalize_customer_worker
Nhiệm vụ: Làm sạch, validate schema và ghi ra Parquet tại tầng Silver
"""
import hashlib
import logging
import awswrangler as wr
import pandas as pd

logger = logging.getLogger()
logger.setLevel(logging.INFO)

SILVER_BUCKET = "data-lakehouse-pzoscg"
SILVER_PREFIX = "silver/customer/"

def clean_phone_number(phone):
    """Chuẩn hóa số điện thoại về định dạng E.164"""
    if pd.isna(phone):
        return None
    cleaned = "".join(filter(str.isdigit, str(phone)))
    if cleaned.startswith("84"):
        return f"+{cleaned}"
    elif cleaned.startswith("0"):
        return f"+84{cleaned[1:]}"
    return f"+{cleaned}"

def lambda_handler(event, context):
    raw_bucket = event["bucket"]
    raw_key = event["key"]
    
    logger.info(f"Bắt đầu chuẩn hóa file: s3://{raw_bucket}/{raw_key}")
    
    # 1. Đọc dữ liệu JSON thô từ S3
    raw_s3_path = f"s3://{raw_bucket}/{raw_key}"
    df = wr.s3.read_json(path=raw_s3_path)
    
    if df.empty:
        logger.warning(f"File rỗng, bỏ qua: {raw_s3_path}")
        return {"status": "SKIPPED_EMPTY"}

    # 2. Thực hiện làm sạch và ép kiểu dữ liệu
    # Loại bỏ khoảng trắng thừa
    df["customer_id"] = df["customer_id"].astype(str).str.strip()
    df["full_name"] = df["name"].astype(str).str.strip().str.title()
    df["email"] = df["email"].astype(str).str.strip().str.lower()
    df["phone"] = df["phone"].apply(clean_phone_number)
    
    # Chuẩn hóa thời gian sang UTC Datetime
    df["registration_date"] = pd.to_datetime(df["created_at"], utc=True)
    
    # Tạo các cột phân vùng
    df["year"] = df["registration_date"].dt.year
    df["month"] = df["registration_date"].dt.month
    
    # Loại bỏ các cột thừa không cần thiết
    columns_to_keep = [
        "customer_id", "full_name", "email", "phone", 
        "registration_date", "year", "month"
    ]
    clean_df = df[columns_to_keep].dropna(subset=["customer_id"])

    # 3. Tạo tên file duy nhất theo MD5 của raw_key để đảm bảo Idempotency
    key_hash = hashlib.md5(raw_key.encode("utf-8")).hexdigest()
    
    # 4. Ghi dữ liệu ra S3 Silver dưới định dạng Snappy Parquet có phân vùng
    silver_s3_path = f"s3://{SILVER_BUCKET}/{SILVER_PREFIX}"
    result = wr.s3.to_parquet(
        df=clean_df,
        path=silver_s3_path,
        dataset=True,
        mode="append",
        database="lakehouse_silver_db",
        table="dim_customer",
        partition_cols=["year", "month"],
        compression="snappy",
        filename_prefix=f"part-{key_hash[:8]}-"
    )
    
    logger.info(f"Đã ghi thành công {len(clean_df)} bản ghi vào Silver: {result['paths']}")
    return {
        "status": "SUCCESS",
        "records_processed": len(clean_df),
        "output_files": result["paths"]
    }
```

---

# IV. Lesson learned / Tổng kết

Qua quá trình xây dựng và tối ưu hóa hệ thống Event-Driven Data Normalization trên thực tế, mình đúc kết được 5 bài học quan trọng:

1. **Bộ 3 quyền hạn IAM bắt buộc cho SQS Event Source Mapping:** Rất nhiều bạn gặp lỗi Lambda không bao giờ được trigger từ SQS dù message liên tục đổ vào queue. Hãy kiểm tra ngay IAM Execution Role của Lambda Controller: nó bắt buộc phải có đầy đủ 3 quyền: `sqs:ReceiveMessage`, `sqs:DeleteMessage` và `sqs:GetQueueAttributes`.
2. **Quy tắc vàng về SQS Visibility Timeout:** Giá trị `VisibilityTimeout` của hàng đợi SQS bắt buộc phải đặt **lớn hơn hoặc bằng 6 lần thời gian Timeout** của hàm Lambda Controller. Nếu các bạn để mặc định 30 giây trong khi Lambda chạy mất 40 giây, SQS sẽ tưởng Lambda bị chết và chuyển tiếp message đó cho một Lambda instance khác xử lý, gây ra hiện tượng nhân đôi dữ liệu!
3. **Giới hạn 15 phút của Lambda và chiến lược phân cấp:** Lambda chỉ phù hợp cho các file thô kích thước vừa và nhỏ (dưới 200 MB). Với các file nén lớn trên 500 MB hoặc batch dồn triệu dòng, Lambda sẽ dễ bị lỗi Out of Memory (OOM) hoặc chạm ngưỡng trần 15 phút timeout. Khi đó, Lambda Controller nên định tuyến để trigger một **AWS Glue Job** hoặc container **Amazon ECS Fargate** thay vì cố ép Lambda xử lý.
4. **Idempotency là chìa khóa của sự bình yên:** Trong kiến trúc hướng sự kiện trên đám mây, triết lý là *"Mọi thứ đều có thể bị thử lại (retry)"*. Việc đặt tên file Parquet tầng Silver dựa trên Hash của file Raw gốc giúp đảm bảo rằng dù network bị ngắt quãng và SQS gửi lại message 3 lần, dữ liệu tại tầng Silver vẫn luôn chính xác tuyệt đối mà không có bản ghi nào bị trùng lặp.
5. **Bước đệm hoàn hảo cho Bài 4:** Đến đây, tầng Silver của chúng ta đã sở hữu dữ liệu sạch sẽ, chuẩn hóa kiểu và tối ưu hóa phân vùng cột. Đây chính là tiền đề hoàn hảo để trong bài viết số 4 tiếp theo, chúng ta sẽ bắt tay vào sử dụng **Apache Spark trên Amazon EMR** để thực hiện các phép tổng hợp Fact/Dimension phức tạp và ghi vào tầng Gold phục vụ Business Intelligence.

Cảm ơn các bạn đã đồng hành cùng chuỗi bài viết. Hẹn gặp lại các bạn trong bài viết tiếp theo!
