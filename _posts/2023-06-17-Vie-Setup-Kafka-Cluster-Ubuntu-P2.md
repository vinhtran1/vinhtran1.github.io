---
title: 'Cài đặt Kafka Cluster trên hệ điều hành Ubuntu - Phần 2/2'
date: 2023-06-17 21:00:00 +0700
categories: [Tutorial]
tags: [Kafka]
pin: false
---
## Giới thiệu

Trong bài trước ([Cài đặt Kafka Cluster trên hệ điều hành Ubuntu - Phần 1/2](https://vinhtran1.github.io/posts/Vie-Setup-Kafka-Cluster-Ubuntu-P1/)) đã dừng lại ở việc setup Zookeeper. Trong bài này chúng ta sẽ tiếp tục setup Kafka và tiến hành chạy thử nhé

## Bước 3: Cấu hình Kafka

Chỉ những server nào được chỉ định cài Kafka thì sẽ thực hiện bước 3 này. Trong bài viết này mặc định 3 server đều có cài đặt Kafka.

### 3.1 Chỉnh sửa file server.properties

Xóa đi file `server.properties` cũ và tạo file mới

```bash
rm config/server.properties && nano config/server.properties
```

Copy và paste nội dung dưới đây. Chú ý một số tham số như:

- Thay đổi các tham số **broker.id**, **listeners** theo từng server
- Tham số **log.dirs** cũng thay đổi về đường dẫn đã tạo `/home/ubuntu/tmp`
- Tham số **zookeeper.connect** điền vào danh sách những server có cài Zookeeper
- Server 1
    
    ```
    broker.id=1
    listeners=PLAINTEXT://kafka1:9092
    num.network.threads=3
    num.io.threads=8
    socket.send.buffer.bytes=102400
    socket.receive.buffer.bytes=102400
    socket.request.max.bytes=104857600
    log.dirs=/home/ubuntu/tmp/kafka-logs
    num.partitions=1
    num.recovery.threads.per.data.dir=1
    offsets.topic.replication.factor=1
    transaction.state.log.replication.factor=1
    transaction.state.log.min.isr=1
    log.retention.hours=168
    log.retention.check.interval.ms=300000
    zookeeper.connect=zookeeper1:2181,zookeeper2:2181,zookeeper3:2181
    zookeeper.connection.timeout.ms=18000
    group.initial.rebalance.delay.ms=0
    ```
    
- Server 2
    
    ```
    broker.id=2
    listeners=PLAINTEXT://kafka2:9092
    num.network.threads=3
    num.io.threads=8
    socket.send.buffer.bytes=102400
    socket.receive.buffer.bytes=102400
    socket.request.max.bytes=104857600
    log.dirs=/home/ubuntu/tmp/kafka-logs
    num.partitions=1
    num.recovery.threads.per.data.dir=1
    offsets.topic.replication.factor=1
    transaction.state.log.replication.factor=1
    transaction.state.log.min.isr=1
    log.retention.hours=168
    log.retention.check.interval.ms=300000
    zookeeper.connect=zookeeper2:2181,zookeeper1:2181,zookeeper3:2181
    zookeeper.connection.timeout.ms=18000
    group.initial.rebalance.delay.ms=0
    ```
    
- Server 3
    
    ```
    broker.id=3
    listeners=PLAINTEXT://kafka3:9092
    num.network.threads=3
    num.io.threads=8
    socket.send.buffer.bytes=102400
    socket.receive.buffer.bytes=102400
    socket.request.max.bytes=104857600
    log.dirs=/home/ubuntu/tmp/kafka-logs
    num.partitions=1
    num.recovery.threads.per.data.dir=1
    offsets.topic.replication.factor=1
    transaction.state.log.replication.factor=1
    transaction.state.log.min.isr=1
    log.retention.hours=168
    log.retention.check.interval.ms=300000
    zookeeper.connect=zookeeper3:2181,zookeeper2:2181,zookeeper1:2181
    zookeeper.connection.timeout.ms=18000
    group.initial.rebalance.delay.ms=0
    ```
    

### 3.2 Bật Kafka

```bash
bin/kafka-server-start.sh config/server.properties
```

Param **-daemon** được thêm vào để Kafka có thể chạy ngầm. Có thể bỏ param này khi cần test, fix bug

```bash
bin/kafka-server-start.sh -daemon config/server.properties
```

Nếu không có lỗi gì xem như chạy thành công và sẵn sàng để sử dụng

### 3.3 (Optional) Setup kafka thành một service

Tương tự như bước 2.5, ta có thể setup Kafka thành một service như sau

Tạo file `/etc/init.d/kafka`

```bash
sudo nano /etc/init.d/kafka
```

Copy và paste nội dung dưới đây

```
#!/bin/bash
#/etc/init.d/kafka
DAEMON_PATH=/home/ubuntu/kafka/bin
DAEMON_NAME=kafka
# Check that networking is up.
#[ ${NETWORKING} = "no" ] && exit 0

PATH=$PATH:$DAEMON_PATH

# See how we were called.
case "$1" in
  start)
        # Start daemon.
        pid=`ps ax | grep -i 'kafka.Kafka' | grep -v grep | awk '{print $1}'`
        if [ -n "$pid" ]
          then
            echo "Kafka is already running"
        else
          echo "Starting $DAEMON_NAME"
          $DAEMON_PATH/kafka-server-start.sh -daemon /home/ubuntu/kafka/config/server.properties
        fi
        ;;
  stop)
        echo "Shutting down $DAEMON_NAME"
        $DAEMON_PATH/kafka-server-stop.sh
        ;;
  restart)
        $0 stop
        sleep 2
        $0 start
        ;;
  status)
        pid=`ps ax | grep -i 'kafka.Kafka' | grep -v grep | awk '{print $1}'`
        if [ -n "$pid" ]
          then
          echo "Kafka is Running as PID: $pid"
        else
          echo "Kafka is not Running"
        fi
        ;;
  *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
esac

exit 0
```

Chạy các câu lệnh dưới đây

```bash
sudo chmod +x /etc/init.d/kafka
sudo chown root:root /etc/init.d/kafka
sudo update-rc.d kafka defaults
```

Tắt Kafka đã chạy ngầm

```bash
bin/kafka-server-stop.sh
```

Bật Kafka

```bash
sudo service kafka start
sudo service kafka status
nc -vz localhost 9092
# Check log
cat /home/ubuntu/kafka/logs/server.log
```

Nếu không có lỗi xem như đã hoàn thành setup một Kafka Cluster

## Bước 4: Những thao tác đầu tiên

### 4.1 Sử dụng Command Line

Tạo topic **vinhdeptrai** tại 1 broker bất kỳ

```bash
bin/kafka-topics.sh --create --topic vinhdeptrai --bootstrap-server localhost:9092 --replication-factor 2 --partitions 1
```

![KafkaCLI1](https://thumbs2.imgbox.com/0e/92/bLoPUwXW_t.png)

Tiếp theo, ta liệt kê topic đã tạo ở một broker khác

```bash
bin/kafka-topics.sh --describe --bootstrap-server localhost:9092
```

![KafkaCLI2](https://thumbs2.imgbox.com/f6/de/MRcv0sxn_t.png)

Xóa topic đã tạo

```bash
bin/kafka-topics.sh --delete --topic vinhdeptrai --bootstrap-server localhost:9092
```

![KafkaCLI3](https://thumbs2.imgbox.com/d4/7d/RqoPJ1k5_t.png)

Kiểm tra lại kết quả đã xóa

```bash
bin/kafka-topics.sh --describe --bootsrap-server localhost:9092
# No Output
```

![KafkaCLI4](https://thumbs2.imgbox.com/12/a8/NHj3mLoA_t.png)

### 4.2 Sử dụng Python

Cài đặt thư viện

```bash
pip install kafka-python
```

Liệt kê topic bằng Python

```python
# filename: main.py
from kafka import KafkaConsumer
consumer = KafkaConsumer(bootstrap_servers=['54.151.243.125:9092', '54.179.131.167:9092', '13.212.222.121:9092'])
print(consumer.topics())
```

Chạy file Python và kiểm tra kết quả

```bash
python3 main.py
# {'vinhdeptrai'}
```

Ta cũng có thể kiểm tra khả năng backup của Kafka bằng cách tắt đi một broker và tạo topic như bình thường

## Kết luận

Tại thời điểm hiện tại, theo như Apache tuyên bố thì từ phiên bản Kafka 2.8 trở đi, quá trình cài đặt đã đơn giản hơn một chút vì không cần cài Zookeeper nữa. 

Xem chi tiết tại link: [https://cwiki.apache.org/confluence/display/KAFKA/KIP-833%3A+Mark+KRaft+as+Production+Ready](https://cwiki.apache.org/confluence/display/KAFKA/KIP-833%3A+Mark+KRaft+as+Production+Ready) 

[https://issues.apache.org/jira/browse/KAFKA-9119](https://issues.apache.org/jira/browse/KAFKA-9119)

Tuy nhiên phải đến phiên bản 4.x thì tính năng này mới hoàn toàn sẵn sàng cho môi trường production. Lúc đó mình sẽ có thêm một bài nữa để hướng dẫn cài đặt Kafka cho phiên bản 4.x

Hy vọng bài viết này có ích cho mọi người