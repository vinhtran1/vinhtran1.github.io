---
title: 'Cài đặt Kafka Cluster trên hệ điều hành Ubuntu - Phần 1/2'
date: 2023-06-17 20:30:00 +0700
categories: [Tutorial]
tags: [Kafka]
pin: false
---

## Giới thiệu

Trong bài viết này mình sẽ không nói về lý thuyết của Apache Kafka mà chỉ tập trung vào từng bước cài đặt một Kafka Cluster với 3 brokers. Phần lý thuyết mình sẽ nói ở một bài viết khác

Có thể tóm tắt quá trình cài đặt Kafka Cluster trong bài viết này thành 3 bước chính:

- Setup các server, thư viện, package, download Kafka
- Cấu hình Zookeeper và bật Zookeeper
- Cấu hình Kafka và bật Kafka

Các phiên bản của các công cụ mình sử dụng trong bài viết này như sau:

- Instance type là t2.small của AWS EC2
- Kafka 3.4.1
- Hệ điều hành Ubuntu 20.04 LTS
- JDK 11.0.19

Để thực hành bài này mọi người nên có một chút kiến thức về AWS nhé

## Bước 1: Cài đặt máy

### 1.1 Tạo Security Group

Tạo một security Group, cần mở các port dưới đây:

- 22: Port để SSH vào các broker
- 2181: port để Kafka giao tiếp với Zookeeper
- 9092: port để Kafka làm việc với Producer và Consumer
- 2888: là port để các broker giao tiếp với nhau trong cùng cluster
- 3888: port để các Zookeeper làm việc và lựa chọn một server làm Leader và các server còn lại làm Follower. Leader node cũng dùng port này để gửi các heartbeat message đến các follower để kiểm tra kết nối

![InboundRules](https://thumbs2.imgbox.com/62/98/0UNY97rD_t.png)

Để tăng độ bảo mật thì mọi người có thể setup các port 2888,3888,2181 nằm chung một subnet thay vì cho phép truy cập từ mọi IP như vậy

### 1.2 Tạo instance AWS EC2

Ở bước này tiến hành tạo 3 instance EC2 có hệ điều hành Ubuntu 20.04 LTS trên AWS sử dụng Security Group vừa tạo ở bước 1.1. Các thao tác tạo instance cụ thể thì mình không nói ở đây. 

Có một lưu ý là cấu hình tối thiểu nên là **t2.small** (1vCPU, 2GB RAM), vì khi cài JDK thì yêu cầu phần cứng tối thiểu là 1GB RAM nên phải chọn cấu hình nhiều hơn 1GB RAM để có thể chạy những tác vụ khác

### 1.3 Cài đặt package

Thực hiện các câu lệnh ở bước này trên cả 3 servers

Update và Upgrade

```bash
sudo apt update && sudo apt upgrade -y
```

Cài đặt các package

```bash
sudo apt install -y wget
```

Cài đặt JDK

```bash
sudo apt -y install default-jdk
```

Kiểm tra phiên bản

```bash
java -version

# openjdk version "11.0.19" 2023-04-18
# OpenJDK Runtime Environment (build 11.0.19+7-post-Ubuntu-0ubuntu120.04.1)
# OpenJDK 64-Bit Server VM (build 11.0.19+7-post-Ubuntu-0ubuntu120.04.1, mixed mode, sharing)
```

### 1.4 Download Kafka

Thực hiện các câu lệnh ở bước này trên cả 3 servers

Download Kafka phiên bản stable hiện tại 3.4.1

```bash
wget https://downloads.apache.org/kafka/3.4.1/kafka_2.13-3.4.1.tgz
```

Giải nén và đổi tên thư mục

```bash
tar -xvzf kafka_2.13-3.4.1.tgz && mv kafka_2.13-3.4.1/ kafka/
```

### 1.5 Thêm địa chỉ IP

Bước này ta sẽ gán các địa chỉ IP thành các host name để có thể dùng trong việc cấu hình Zookeeper và Kafka

Giả sử địa chỉ IP của các server là:

- Server 1: 54.151.243.125
- Server 2: 54.179.131.167
- Server 3: 13.212.222.12

Ta sẽ thực hiện câu lệnh sau để thêm vào file `/etc/hosts`:

- Server 1
    
    ```bash
    echo "0.0.0.0 kafka1
    0.0.0.0 zookeeper1
    54.179.131.167 kafka2
    54.179.131.167 zookeeper2
    13.212.222.121 kafka3
    13.212.222.121 zookeeper3" | sudo tee --append /etc/hosts
    ```
    
- Server 2
    
    ```bash
    echo "54.151.243.125 kafka1
    54.151.243.125 zookeeper1
    0.0.0.0 kafka2
    0.0.0.0 zookeeper2
    13.212.222.121 kafka3
    13.212.222.121 zookeeper3" | sudo tee --append /etc/hosts
    ```
    
- Server 3
    
    ```bash
    echo "54.151.243.125 kafka1
    54.151.243.125 zookeeper1
    54.179.131.167 kafka2
    54.179.131.167 zookeeper2
    0.0.0.0 kafka3
    0.0.0.0 zookeeper3" | sudo tee --append /etc/hosts
    ```
    

**Lưu ý:** Không nhất thiết Zookeeper và Kafka luôn luôn có cùng địa chỉ IP. Nếu chúng được cài đặt trên các server khác nhau thì cần tùy chỉnh lại địa chỉ IP cho tương ứng

## Bước 2: Cấu hình Zookeeper

Trong bài viết này mặc định cả 3 servers đều cài Zookpeer. Tuy nhiên như đã nói, Zookeeper và Kafka có thể ở trên 2 server khác nhau hoặc giống nhau, không nhất thiết tất cả server được tạo ra đều phải cài Zookeeper. Vì vậy, trong thực tế, chỉ những server nào được chỉ định cài đặt Zookeeper thì thực hiện Bước 2 này. 

### 2.1 Tạo thư mục tmp/zookeeper

```bash
mkdir -p tmp/zookeeper
```

thư mục này sẽ là **dataDir** trong file `zookeeper.properties`

### 2.2 Tạo file myid vào thư mục tmp/zookeeper

`myid` là file chứa một số nguyên làm id cho broker. Các server khác nhau của cùng 1 cluster kafka sẽ có các id khác nhau

- Server 1
    
    ```bash
    echo "1" > /home/ubuntu/tmp/zookeeper/myid
    ```
    
- Server 2
    
    ```bash
    echo "2" > /home/ubuntu/tmp/zookeeper/myid
    ```
    
- Server 3
    
    ```bash
    echo "3" > /home/ubuntu/tmp/zookeeper/myid
    ```
    

### 2.3 Chỉnh sửa file zookeeper.properties

Di chuyển vào trong thư mục `kafka/`

```bash
cd kafka/
```

Xóa đi file `zookeeper.properties` cũ và tạo file mới

```bash
rm config/zookeeper.properties && nano config/zookeeper.properties
```

Copy và Paste nội dung dưới đây

```
dataDir=/home/ubuntu/tmp/zookeeper
clientPort=2181
maxClientCnxns=128
tickTime=2000
initLimit=10
syncLimit=5
server.1=zookeeper1:2888:3888
server.2=zookeeper2:2888:3888
server.3=zookeeper3:2888:3888
```

Bấm Ctrl + X → bấm Y → bấm Enter để thoát

Giả sử Server 1 không được chỉ định cài Zookeeper thì ta sẽ bỏ dòng **server.1=zookeeper1:2888:3888** ra khỏi file

### 2.4 Bật Zookeeper

Chạy câu lệnh dưới đây để bật Zookeeper

```bash
bin/zookeeper-server-start.sh config/zookeeper.properties
```

Param **-daemon** được thêm vào để Zookeeper có thể chạy ngầm. Có thể bỏ param này khi cần test, fix bug

```bash
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
```

### 2.5 (Optional) Setup Zookeeper thành một service

Bước này nhằm mục đích làm cho việc khởi tạo Zookeeper thành một service. Nó có một số lợi ích như:

- Khi server có sự cố cần reboot thì sẽ tự động bật lại Zookeeper thay vì phải bật thủ công
- Dễ quản lý các tác vụ chạy ngầm

Tạo file `/etc/init.d/zookeeper`

```bash
sudo nano /etc/init.d/zookeeper
```

Copy và paste nội dung dưới đây

```
#!/bin/sh
#
# zookeeper          Start/Stop zookeeper
#
# chkconfig: - 99 10
# description: Standard script to start and stop zookeeper

DAEMON_PATH=/home/ubuntu/kafka/bin
DAEMON_NAME=zookeeper

PATH=$PATH:$DAEMON_PATH

# See how we were called.
case "$1" in
  start)
        # Start daemon.
        pid=`ps ax | grep -i 'org.apache.zookeeper' | grep -v grep | awk '{print $1}'`
        if [ -n "$pid" ]
          then
            echo "Zookeeper is already running";
        else
          echo "Starting $DAEMON_NAME";
          $DAEMON_PATH/zookeeper-server-start.sh -daemon /home/ubuntu/kafka/config/zookeeper.properties
        fi
        ;;
  stop)
        echo "Shutting down $DAEMON_NAME";
        $DAEMON_PATH/zookeeper-server-stop.sh
        ;;
  restart)
        $0 stop
        sleep 2
        $0 start
        ;;
  status)
        pid=`ps ax | grep -i 'org.apache.zookeeper' | grep -v grep | awk '{print $1}'`
        if [ -n "$pid" ]
          then
          echo "Zookeeper is Running as PID: $pid"
        else
          echo "Zookeeper is not Running"
        fi
        ;;
  *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
esac

exit 0
```

Sau khi tạo file, chạy lần lượt 3 câu lệnh sau đây

```bash
sudo chmod +x /etc/init.d/zookeeper
sudo chown root:root /etc/init.d/zookeeper
sudo update-rc.d zookeeper defaults
```

Trước khi bật Zookeeper, cần phải tắt Zookeeper đã chạy ngầm ở bước 1.4

```bash
bin/zookeeper-server-stop.sh
```

Sau đó bật lại bằng lệnh service

```bash
sudo service zookeeper start
sudo service zookeeper status
```

Như vậy là đã hoàn thành cài đặt Zookeeper. Mình xin kết thúc phần 1 tại đây, sang phần tiếp theo sẽ tiếp tục setup Kafka và bắt đầu sử dụng nhé