---
title: 'Change Data Capture (CDC) toàn tập: Đồng bộ dữ liệu Real-time với Debezium và Kafka Connect'
date: 2026-09-24 09:45:00 +0700
categories: [Architecture, Data Engineering]
tags: [CDC, Debezium, Apache Kafka, Kafka Connect, PostgreSQL, Event-Driven]
keywords: [CDC, Debezium, Kafka Connect, PostgreSQL, Event-Driven]
pin: false
image:
  path: /assets/img/posts/2026/change-data-capture-debezium-kafka-connect/cover.webp
  alt: 'Kiến trúc Change Data Capture (CDC) với Debezium, Kafka Connect và PostgreSQL'
---

# I. Dẫn nhập

Trong các hệ thống phân tán và kiến trúc microservices hiện đại, nhu cầu đồng bộ dữ liệu thời gian thực giữa các kho lưu trữ là bài toán mà hầu như bất kỳ kỹ sư dữ liệu hay backend developer nào cũng từng đối mặt. Các bạn thử hình dung: Một nghiệp vụ thay đổi đơn hàng (order) diễn ra trên cơ sở dữ liệu giao dịch chính (OLTP Database như PostgreSQL), ngay lập tức ta cần:
1. Đồng bộ bản ghi mới sang Data Warehouse (BigQuery, Snowflake, ClickHouse) để chạy báo cáo phân tích.
2. Cập nhật lại chỉ mục tìm kiếm trên Elasticsearch.
3. Làm mới hoặc xóa cache tương ứng trên Redis (Cache Invalidation).
4. Bắn thông báo đẩy (Push Notification) hoặc WebSocket về ứng dụng di động của khách hàng.

Khi mới bắt tay giải quyết bài toán này, giải pháp trực quan và phổ biến nhất mà các team thường nghĩ đến là **Polling (Truy vấn định kỳ)**:
```sql
SELECT * FROM orders WHERE updated_at > :last_poll_timestamp ORDER BY updated_at ASC;
```

Tuy nhiên, trong quá trình đi làm thực tế, mình nhận thấy phương pháp Polling bộc lộ những nhược điểm chí mạng:
- **Tải nặng lên database nguồn:** Việc định kỳ vài giây một lần quét bảng (dù có index trên `updated_at`) vẫn tiêu tốn CPU, tranh chấp I/O đĩa và có thể gây lock bảng khi lượng ghi lớn.
- **Bỏ lỡ sự kiện `DELETE`:** Nếu một bản ghi bị `DELETE` vật lý khỏi bảng, câu truy vấn `SELECT` kiểm tra `updated_at` hoàn toàn không thể bắt được bản ghi đó, dẫn đến dữ liệu ở các hệ thống đích bị rác và lệch chuẩn (data drift).
- **Độ trễ (Latency) cố hữu:** Polling luôn có khoảng trễ giữa các chu kỳ. Nếu giảm chu kỳ xuống dưới 1 giây thì database sập vì quá tải, còn nếu tăng chu kỳ lên vài phút thì mất tính chất real-time.
- **Trùng lặp sự kiện khi update nhanh:** Nếu một bản ghi được cập nhật 3 lần giữa 2 lần poll, hệ thống đích chỉ thấy trạng thái cuối cùng mà mất đi 2 trạng thái chuyển đổi trung gian.

Để vượt qua những giới hạn này, **Change Data Capture (CDC) dựa trên Transaction Log** ra đời. Thay vì hỏi cơ sở dữ liệu *"có gì mới không?"*, CDC biến cơ sở dữ liệu thành một luồng sự kiện (event stream) liên tục và tức thì.

Trong bài viết này, mình sẽ cùng các bạn tìm hiểu tường tận nguyên lý Log-based CDC, kiến trúc kết hợp giữa **Debezium**, **Kafka Connect**, **Apache Kafka** và **PostgreSQL**, đồng thời dựng một hệ thống hoàn chỉnh từ file `docker-compose.yml` đến kiểm thử bắt sự kiện INSERT, UPDATE, DELETE trong thực tế!

---

# II. Kiến trúc / Nguyên lý

### 1. Cơ chế Log-based CDC: Trái tim Write-Ahead Log (WAL)

Bản chất của Log-based CDC nằm ở việc khai thác nhật ký giao dịch ghi trước (**Write-Ahead Log - WAL** trong PostgreSQL, hay **Binary Log - Binlog** trong MySQL). 

Mọi hệ thống RDBMS tuân thủ chuẩn ACID đều bắt buộc phải ghi nhận thao tác thay đổi vào log file tuần tự trên đĩa cứng trước khi thực sự áp dụng thay đổi vào các trang dữ liệu (data pages) trong bộ nhớ và đĩa.

```
Client Commit Transaction
       │
       ▼
┌──────────────────────────────────────────────┐
│ PostgreSQL WAL (Write-Ahead Logging)          │
│ [LSN 0/1A01: INSERT orders]                  │
│ [LSN 0/1A02: UPDATE orders]                  │
│ [LSN 0/1A03: DELETE orders]                  │
└──────────────────────┬───────────────────────┘
                       │ Logical Decoding via pgoutput
                       ▼
┌──────────────────────────────────────────────┐
│ Debezium PostgreSQL Connector                │
│ (Reads logical replication stream)           │
└──────────────────────┬───────────────────────┘
                       │ JSON / Avro Events
                       ▼
┌──────────────────────────────────────────────┐
│ Apache Kafka Cluster (Partitioned by PK)     │
│ Topic: postgres.public.orders                │
└──────────────────────────────────────────────┘
```

Trong PostgreSQL, cơ chế này được thực hiện thông qua **Logical Decoding**:
- Tham số cấu hình `wal_level = logical` yêu cầu Postgres ghi đầy đủ thông tin hàng thay đổi (dữ liệu trước và sau biến đổi) vào WAL.
- Plugin giải mã chuẩn tích hợp sẵn trong Postgres từ bản 10 trở lên là **`pgoutput`**, cho phép biến đổi stream nhị phân của WAL thành các thông điệp logic rõ ràng.
- **Replication Slot:** Đảm bảo Postgres ghi nhớ vị trí LSN (Log Sequence Number) mà consumer đã đọc tới. Database sẽ không xóa các file WAL cũ chừng nào replication slot chưa xác nhận (ACK).

> **Công thức ghi nhớ:**  
> Log-based CDC = Đọc trực tiếp WAL + Logical Decoding (`pgoutput`) + Non-blocking Zero-overhead lên Database

### 2. Kiến trúc Debezium và Kafka Connect

Debezium là một nền tảng mã nguồn mở phân tán chuyên biệt cho Change Data Capture. Thay vì tự viết code kết nối socket để đọc WAL, Debezium đóng gói toàn bộ logic kết nối, snapshot ban đầu, quản lý offset và chuyển đổi schema thành một connector plugin chuẩn chạy trên nền **Kafka Connect**.

Hệ sinh thái bao gồm 4 thành phần chính:
1. **Source Database (PostgreSQL):** Chứa dữ liệu nghiệp vụ, được cấu hình mở logical replication.
2. **Kafka Connect Cluster:** Một runtime JVM phân tán chịu trách nhiệm quản lý vòng đời của các Connector. Debezium PostgreSQL Connector chạy bên trong Kafka Connect, kết nối vào Postgres như một replication client.
3. **Apache Kafka Brokers:** Lưu trữ các message stream vào các topic. Theo mặc định, Debezium sẽ định tuyến mỗi bảng thành một Kafka topic riêng theo định dạng: `<serverName>.<schemaName>.<tableName>` (ví dụ: `dbserver1.public.orders`). Khóa của message (Kafka message key) chính là Primary Key của bản ghi, đảm bảo thứ tự sự kiện trên cùng một thực thể luôn được bảo toàn tuần tự trên một partition.
4. **Downstream Consumers:** Các microservice, search engine, cache engine hoặc sink connectors (Kafka Connect JDBC Sink, Elasticsearch Sink, S3 Sink) lắng nghe topic để cập nhật dữ liệu.

### 3. Giải mã cấu trúc Debezium Event Payload

Một trong những điểm mạnh nhất của Debezium là cấu trúc payload sự kiện vô cùng rõ ràng và chuẩn hóa. Khi một giao dịch diễn ra, Debezium sinh ra một JSON payload với 4 trường cốt lõi:

```json
{
  "schema": { ... },
  "payload": {
    "before": {
      "id": 101,
      "customer_id": "CUST-01",
      "total_amount": 150.00,
      "status": "PENDING"
    },
    "after": {
      "id": 101,
      "customer_id": "CUST-01",
      "total_amount": 150.00,
      "status": "COMPLETED"
    },
    "source": {
      "version": "2.5.0.Final",
      "connector": "postgresql",
      "name": "dbserver1",
      "ts_ms": 1727163900000,
      "snapshot": "false",
      "db": "inventory",
      "sequence": "[\"24021200\",\"24021200\"]",
      "schema": "public",
      "table": "orders",
      "txId": 589,
      "lsn": 24021200
    },
    "op": "u",
    "ts_ms": 1727163900500
  }
}
```

Hãy phân tích các trường này:
- **`op` (Operation Type):** Loại thao tác trong cơ sở dữ liệu:
  - `c`: Create (INSERT). Trường `before` sẽ là `null`, trường `after` chứa dữ liệu mới.
  - `u`: Update (UPDATE). Cả `before` và `after` đều có giá trị (yêu cầu đặt `REPLICA IDENTITY FULL` trên bảng nếu muốn thấy toàn bộ trạng thái cũ).
  - `d`: Delete (DELETE). Trường `before` chứa dữ liệu trước khi xóa, trường `after` là `null`.
  - `r`: Read (Initial Snapshot). Đọc trong giai đoạn bootstrap ban đầu.
- **`before` / `after`:** Snapshot của hàng trước và sau thao tác.
- **`source`:** Toàn bộ siêu dữ liệu giao dịch: tên database, tên bảng, commit timestamp (`ts_ms`), mã giao dịch (`txId`), và vị trí chính xác trong WAL (`lsn`).
- **`ts_ms`:** Thời điểm Debezium xử lý sự kiện, cho phép tính toán độ trễ (Replication Lag = `ts_ms` của Debezium - `ts_ms` của source).

---

# III. Cài đặt / Hands-on code

Bây giờ, chúng ta sẽ bắt tay vào thực hành dựng toàn bộ luồng CDC từ đầu đến cuối trên máy tính của các bạn.

### Bước 1: Khởi tạo cụm dịch vụ với Docker Compose

Tạo file `docker-compose.yml` gồm PostgreSQL (đã cấu hình sẵn logical replication), Apache Zookeeper, Apache Kafka và Kafka Connect với plugin Debezium PostgreSQL tích hợp sẵn:

```yaml
# filename: docker-compose.yml
version: '3.8'

services:
  postgres:
    image: debezium/postgres:16-alpine
    container_name: postgres
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=inventory
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"
      - "-c"
      - "max_wal_senders=4"
      - "-c"
      - "max_replication_slots=4"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  debezium:
    image: debezium/connect:2.5
    container_name: debezium
    depends_on:
      - kafka
      - postgres
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: my_connect_configs
      OFFSET_STORAGE_TOPIC: my_connect_offsets
      STATUS_STORAGE_TOPIC: my_connect_statuses
      CONFIG_STORAGE_REPLICATION_FACTOR: 1
      OFFSET_STORAGE_REPLICATION_FACTOR: 1
      STATUS_STORAGE_REPLICATION_FACTOR: 1
```

Khởi chạy cụm dịch vụ bằng lệnh:
```bash
docker compose up -d
```

### Bước 2: Tạo bảng dữ liệu và cấu hình Replica Identity trên PostgreSQL

Kết nối vào container PostgreSQL để tạo bảng `customers`:

```sql
-- filename: init.sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Quan trọng: Đặt REPLICA IDENTITY FULL để Debezium nhận trọn vẹn giá trị cột 'before' khi UPDATE/DELETE
ALTER TABLE customers REPLICA IDENTITY FULL;

-- Chèn dữ liệu mẫu ban đầu
INSERT INTO customers (first_name, last_name, email)
VALUES ('Vinh', 'Tran', 'vinh@example.com'),
       ('Anh', 'Nguyen', 'anh@example.com');
```

**Lưu ý:** Theo mặc định, PostgreSQL sử dụng `REPLICA IDENTITY DEFAULT`, nghĩa là khi có thao tác UPDATE hoặc DELETE, Postgres chỉ ghi giá trị khóa chính (Primary Key) vào WAL cho phần `before`. Khi ta chuyển sang `FULL`, toàn bộ giá trị cũ của tất cả các cột đều được ghi vào WAL, giúp consumer dễ dàng so sánh sự thay đổi giá trị của từng field.

### Bước 3: Đăng ký Debezium Connector qua REST API

Kafka Connect cung cấp một REST API chuẩn chạy tại cổng `8083`. Ta gửi request HTTP POST để kích hoạt connector giám sát bảng `customers`:

```bash
curl -i -X POST http://localhost:8083/connectors \
  -H "Accept:application/json" \
  -H "Content-Type:application/json" \
  -d '{
    "name": "inventory-connector",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "tasks.max": "1",
      "plugin.name": "pgoutput",
      "database.hostname": "postgres",
      "database.port": "5432",
      "database.user": "postgres",
      "database.password": "postgres",
      "database.dbname": "inventory",
      "database.server.name": "dbserver1",
      "topic.prefix": "dbserver1",
      "table.include.list": "public.customers",
      "schema.history.internal.kafka.bootstrap.servers": "kafka:29092",
      "schema.history.internal.kafka.topic": "schema-changes.inventory"
    }
  }'
```

Kiểm tra trạng thái connector hoạt động:
```bash
curl -s http://localhost:8083/connectors/inventory-connector/status | jq .
```
Nếu `state` của connector và tasks đều là `RUNNING`, hệ thống đã sẵn sàng stream dữ liệu!

### Bước 4: Kiểm chứng luồng dữ liệu thời gian thực trên Kafka Topic

Lắng nghe topic `dbserver1.public.customers` từ bên trong container Kafka:

```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic dbserver1.public.customers \
  --from-beginning
```

Bây giờ, tại một terminal khác, hãy thử thực hiện lệnh UPDATE và DELETE trong PostgreSQL:

```sql
-- Cập nhật email của khách hàng ID = 1
UPDATE customers SET email = 'vinh.tran.tech@example.com' WHERE id = 1;

-- Xóa khách hàng ID = 2
DELETE FROM customers WHERE id = 2;
```

Ngay lập tức trên console consumer, các bạn sẽ thấy 2 message JSON đổ về với độ trễ chỉ dưới 10 mili-giây:
- Thao tác UPDATE xuất hiện với `"op": "u"`, trường `before.email` là `'vinh@example.com'` và `after.email` là `'vinh.tran.tech@example.com'`.
- Thao tác DELETE xuất hiện với `"op": "d"`, trường `before` chứa toàn bộ dữ liệu của khách hàng số 2, và theo sau là một **Tombstone Message** (message có value là `null`) để hỗ trợ cơ chế Log Compaction trong Kafka.

---

# IV. Lesson learned / Tổng kết

Change Data Capture với Debezium và Kafka Connect thực sự là một bước nhảy vọt so với phương pháp Polling truyền thống, giúp kiến trúc dữ liệu trở nên thanh thoát, chịu tải cao và gần như triệt tiêu hoàn toàn độ trễ. 

Tuy nhiên, khi đưa giải pháp này vào môi trường Production thực tế, mình đúc kết 4 bài học xương máu mà các bạn nhất định phải lưu tâm:

1. **Hiểm họa tràn ổ cứng do Replication Slot (Replication Slot Disk Consumption):**
   - Khi một Replication Slot được kích hoạt, PostgreSQL có trách nhiệm giữ lại tất cả các file WAL chưa được consumer xác nhận (ACK).
   - Nếu cụm Kafka Connect bị tắt hoặc network bị đứt trong nhiều giờ mà ứng dụng vẫn liên tục ghi dữ liệu vào Postgres, các file WAL trong thư mục `pg_wal` sẽ phình to không giới hạn cho tới khi ổ cứng đầy 100%, kéo theo việc database bị crash!
   - **Giải pháp:** Thiết lập tham số `max_slot_wal_keep_size` trong `postgresql.conf` (ví dụ `10GB`) để giới hạn dung lượng tối đa mà slot được giữ, đồng thời cài đặt cảnh báo (alert) giám sát dung lượng replication slot qua view `pg_replication_slots`.

2. **Chiến lược Snapshot ban đầu cho bảng lớn (Initial Snapshotting):**
   - Khi connector khởi động lần đầu, Debezium sẽ đọc toàn bộ dữ liệu hiện có trong bảng (Initial Snapshot) trước khi chuyển sang đọc stream WAL.
   - Với những bảng chứa hàng trăm triệu dòng, quá trình snapshot có thể gây tăng đột biến I/O và tiêu tốn nhiều giờ. Hãy tận dụng tính năng **Incremental Snapshot** của Debezium (`signal.data.collection`) để chia nhỏ quá trình snapshot thành từng batch song song với quá trình capture WAL mà không cần khóa bảng.

3. **Xử lý tiến hóa Schema (Schema Drift & Evolution):**
   - Trong quá trình phát triển, các bạn sẽ thường xuyên chạy migration (`ALTER TABLE ADD COLUMN`, `DROP COLUMN`).
   - Cần cấu hình Debezium đi kèm với **Confluent Schema Registry** và sử dụng định dạng serialization chuẩn như **Apache Avro** hoặc **Protobuf** thay vì JSON thô. Điều này đảm bảo tính tương thích xuôi và ngược (Backward/Forward Compatibility) cho downstream consumers.

4. **Định tuyến Partition Key để đảm bảo thứ tự sự kiện:**
   - Trong Kafka, thứ tự sự kiện chỉ được bảo đảm tuyệt đối trên cùng một Partition.
   - Debezium tự động chọn Primary Key làm message key, giúp mọi thao tác INSERT, UPDATE, DELETE trên cùng một bản ghi luôn đi vào cùng một partition. Nếu bảng của các bạn không có Primary Key, Debezium sẽ đẩy ngẫu nhiên vào các partition khác nhau dẫn đến nguy cơ consumer xử lý sai thứ tự (xử lý DELETE trước UPDATE).

Hy vọng bài viết thực chiến này đã giúp các bạn hiểu rõ từ bản chất lý thuyết đến cách vận hành thực tế của kiến trúc CDC với Debezium và Kafka Connect. Trong bài viết tiếp theo, mình sẽ chia sẻ về một chủ đề thú vị không kém: Cách tận dụng chính PostgreSQL với bảng UNLOGGED và JSONB để làm Cache hiệu năng cao thay thế cho Redis!
