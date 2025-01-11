---
title: 'Dùng PostgreSQL để làm một Message Queue'
date: 2023-05-26 23:25:00 +0700
categories: [Tutorial]
tags: [PostgreSQL]
pin: false
---

## Giới thiệu

Khi nhắc đến các ứng dụng message queue, nhiều người có thể nghĩ ngay tới `RabbitMQ`, `Apache Kafka`, `Apache ActiveMQ`,… là những công cụ được hỗ trợ rất mạnh mẽ. Nhưng giả sử số lượng message không quá lớn, không có nhu cầu scale quá phức tạp, cũng như không muốn tốn thêm chi phí để setup server cho các ứng dụng bên thứ ba thì tận dụng PostgreSQL để làm một message queue cũng là một cách đáng cân nhắc

## Tóm tắt cách hoạt động

Nếu bạn có tìm hiểu qua về Design Pattern thì chức năng NOTIFY, LISTEN của PostgreSQL hoạt động giống như `Observer Pattern`. Nhưng nếu không biết thì bạn có thể tham khảo hình dưới đây

![notify_listen_pg.png](https://images2.imgbox.com/a4/92/ZXAhbo1w_o.png)

- Ở phía Notifier: có một service sẽ insert message vào một bảng trong PostgreSQL. Sau đó sẽ có một trigger thực hiện lấy dữ liệu đó ra và thực hiện câu lệnh NOTIFY vào một channel
- Ở phía Listener: các service sẽ chờ để “listen” một channel cụ thể xem có bất kỳ message nào được notify không. Nếu có sẽ lấy ra và xử lý. Các service listen cùng một channel đều sẽ nhận được message này

## Hướng dẫn cài đặt

### Tạo bảng chứa message

Đầu tiên ta tạo một bảng chứa 2 cột ID và message

```sql
CREATE TABLE public.message_queue (
	id uuid NOT NULL DEFAULT gen_random_uuid(),
	message jsonb NULL,
	created_at timestamptz NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Tạo function

Tạo một function để notify mỗi khi có dữ liệu mới được insert vào bảng `message_queue`.

Giả sử trong function này ta sẽ notify vào channel `sample_channel`

```sql
CREATE OR REPLACE FUNCTION public.notify_function()
 RETURNS trigger
 LANGUAGE plpgsql
AS $function$
      BEGIN
      PERFORM pg_notify('sample_channel', row_to_json(NEW)::text );
        RETURN NEW;
      END;
  $function$
;
```

Như vậy là đã setup xong message queue bằng PostgreSQL. Tiếp theo ta sẽ viết code cho producer và consumer

### Tạo trigger

Ở bước này, ta sẽ tạo một trigger để hoạt động mỗi khi có message mới được insert vào bảng `message_queue`

```sql
CREATE TRIGGER message_queue_trigger
AFTER INSERT
ON public.message_queue 
FOR EACH ROW EXECUTE FUNCTION public.notify_function();
```

### Viết code listener

Trước hết, cần cài đặt thư viện psycopg2

```bash
pip install psycopg2-binary
```

Đoạn code Python để consume message như sau:

```python
# filename: listener.py
import asyncio
import psycopg2
import time

conn = psycopg2.connect(host="localhost", dbname="postgres", user="postgres", password="", port=5432)
conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)

cursor = conn.cursor()
cursor.execute(f"LISTEN sample_channel;")

def handle_notify():
    conn.poll()
    for notify in conn.notifies:
        time.sleep(1)
        print(notify.payload)
    conn.notifies.clear()

# It works with uvloop too:
# import uvloop
# loop = uvloop.new_event_loop()
# asyncio.set_event_loop(loop)

loop = asyncio.get_event_loop()
loop.add_reader(conn, handle_notify)
loop.run_forever()
```

### Viết code notifier

```python
# filename: notifier.py
import time
import psycopg2
import json

conn = psycopg2.connect(host="localhost", dbname="postgres", user="postgres", password="", port=5432)

cursor = conn.cursor()
conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)

i = 0
while True:
    val = time.time()
    payload_dictionary = {
        "message": f"message number {i}"
    }
    payload_str = json.dumps(payload_dictionary)
    print(payload_str)
    cursor.execute(f"""
                    INSERT INTO public.message_queue
                    (message)
                    VALUES('{payload_str}');
                    """)
    i = i + 1
    time.sleep(1)
```

## Chạy thực nghiệm

Đầu tiên bật listener service

```bash
python3 listener.py
```

Tiếp theo là bật notifier service

```bash
python3 notifier.py
```

Nếu chạy thành công, thì kết quả của notifer sẽ được in ra như sau:

```
Inserting message {"message": "message number 0"} to message table
Inserting message {"message": "message number 1"} to message table
Inserting message {"message": "message number 2"} to message table
Inserting message {"message": "message number 3"} to message table
Inserting message {"message": "message number 4"} to message table
Inserting message {"message": "message number 5"} to message table
```

Và kết quả tương ứng hiển thị ở listener:

```
{"id":"4d13545c-2ebc-471f-8f0e-21a570807fee","message":{"message": "message number 0"},"created_at":"2023-05-26T15:17:04.184093+00:00"}
{"id":"cfca6e21-8ea5-42a4-8beb-bc68d81acadd","message":{"message": "message number 1"},"created_at":"2023-05-26T15:17:05.188328+00:00"}
{"id":"e4fea59e-0037-4a67-8510-573051fbd00a","message":{"message": "message number 2"},"created_at":"2023-05-26T15:17:06.191606+00:00"}
{"id":"64e9f5aa-8011-43b4-911f-7378f7bbf476","message":{"message": "message number 3"},"created_at":"2023-05-26T15:17:07.19488+00:00"}
{"id":"eb8e0ce9-17a9-4198-9394-9b914e0d35db","message":{"message": "message number 4"},"created_at":"2023-05-26T15:17:08.197062+00:00"}
{"id":"e480bb79-38cb-45e5-bd8b-65d7b7ae03c9","message":{"message": "message number 5"},"created_at":"2023-05-26T15:17:09.200687+00:00"}
```

## Đánh giá về NOTIFY-LISTEN trong PostgreSQL

### Ưu điểm

- Dễ cài đặt: Việc cài đặt hoàn toàn được thực hiện trên PostgreSQL mà không cần cài thêm bất kỳ ứng dụng hay extension nào
- Không tốn thêm chi phí để setup server riêng: Có thể tận dụng PostgreSQL đang có của ứng dụng để cài đặt mà không cần mất phí thuê mới server
- Miễn phí: Đây là tính năng có sẵn trên PostgreSQL

### Nhược điểm

- Chỉ thích hợp cho nhu cầu cơ bản: Với các hệ thống có lượng traffic lớn thì nên chuyển sang dùng các ứng dụng chuyên dùng như RabbitMQ hay Kafka
- Giới hạn kích thước mỗi message chỉ là 8000 bytes so với 1MB của Kafka hay 128MB của RabbitMQ
- Khó Scale