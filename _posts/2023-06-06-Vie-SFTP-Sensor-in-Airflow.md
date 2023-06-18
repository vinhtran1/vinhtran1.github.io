---
title: 'Dùng Airflow để phát hiện có file mới trên SFTP Server: SFTP Sensor'
date: 2023-06-06 15:10:00 +0700
categories: [Tutorial]
tags: [Airflow]
pin: false
---

## Giới thiệu

Airflow là một công cụ lập lịch và quản lý các Job hỗ trợ ngôn ngữ lập trình Python rất phổ biến. Nhưng ngoài ra, Airflow cũng có thể xử lý được một số tác vụ cần xử lý nhanh chóng với độ trễ nhỏ bằng các Sensor. Trong bài viết này sẽ giới thiệu và hướng dẫn cài đặt SFTP Sensor trong Airflow.

Tính năng này được dùng để theo dõi thường xuyên hệ thống SFTP Server để phát hiện xem có bất kỳ file nào mới được upload lên hay không. Nếu có thì sẽ chuyển sang task tiếp theo để xử lý

## Cấu hình

Trong bài này sẽ sử dụng những công nghệ sau

- Docker version 24.0.0
- Python 3.8
- Airflow version 2.6.1

## Bước 0: Setup hệ thống Airflow, SFTP

### Setup Airflow

Để cài đặt Airflow mọi người dựa vào link này nhé: [https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)

### Setup SFTP

Để cài đặt SFTP mọi người dựa vào link này nhé: [https://hub.docker.com/r/atmoz/sftp](https://hub.docker.com/r/atmoz/sftp)

Nếu mọi người chưa biết dùng công cụ gì để tương tác với SFTP Server thì có một gợi ý là dùng **FileZilla**. Công cụ này hoạt động được trên các hệ điều hành phổ biến Windows, MacOS, Ubuntu: [https://filezilla-project.org/](https://filezilla-project.org/)

### Cài đặt thư viện

Để hoạt động được thì cần cài thêm thư viện như sau

```bash
pip install apache-airflow==2.6.1 apache-airflow-providers-sftp==4.3.0
```
Nếu đã cài đặt Airflow phiên bản mới nhất bằng Docker thì có thể bỏ qua bước này.

Như vậy là tạm thời xong setup các công cụ cần thiết.

## Bước 1: Tạo connection tới SFTP

Vào **Admin** → **Connections** → Bấm **Add a new Record** để tạo connection

![AddConnection.png](https://images2.imgbox.com/79/65/iMEx8dHb_o.png)

Chọn **Connection Type** là SFTP, sau đó điền vào các thông tin của SFTP

Sau khi điền xong, bấm **Test** để kiểm tra kết nối. Sau đó bấm **Save** để lưu kết nối

![ConnectionSFTP.png](https://images2.imgbox.com/c7/75/rD9BzwuV_o.png)

## Bước 2: Viết code Python

Copy đoạn code sau và bỏ vào thư mục `dags`

```python
from airflow.models import DAG
from airflow.providers.sftp.sensors.sftp import SFTPSensor
from airflow.operators.python_operator import PythonOperator
from airflow.operators.trigger_dagrun import TriggerDagRunOperator
from airflow.utils.dates import days_ago
import logging
from datetime import datetime

def do_somthing(**context):
    logging.info("New file uploaded! Do something here...")

with DAG("sftp_sensor_dag",
         schedule_interval=None,
         max_active_runs=1,
         start_date=days_ago(2)) as dag:
    sensor = SFTPSensor(task_id="wait_for_new_file",
                        sftp_conn_id="sftp_connection",
                        path="/home/files/input_data",
                        file_pattern='*.csv',
                        newer_than=datetime.now(),
                        poke_interval=5,
                        timeout=10,
                        silent_fail=True,
                        soft_fail=True)
    processor = PythonOperator(
        task_id='process_file',
        python_callable=do_somthing
    )
    trigger = TriggerDagRunOperator(
        task_id='trigger_this_dag',
        trigger_dag_id='sftp_sensor_dag',
        trigger_rule='all_done'
    )
    sensor >> processor >> trigger
```

Sau đó sẽ chờ Airflow load DAG này

Chú ý một số param như sau:

- **sftp_conn_id**: là Connection Id của Connection mà chúng ta đã tạo ở bước 1
- **path**: là đường dẫn đến thư mục mà sensor hoạt động. Sensor sẽ liên tục kiểm tra đường dẫn thư mục này để tìm file mới
- **file_pattern**: là format của filename, dùng để xác định xem file vừa được upload có phải là file cần xử lý không. Pattern này theo chuẩn **fnmatch**
- **newer_than**: kiểu dữ liệu datetime, là mốc thời gian dùng để so sánh với thời gian upload file lên SFTP Server. Nếu file nào có thời gian lớn hơn param này sẽ kết thúc task **sensor** và chuyển sang task **processor** để thực hiện xử lý dữ liệu
- **poke_interval**: Là thời gian (tính bằng giây) giữa 2 lần quét thư mục
- **timeout**: là thời gian (tính bằng giây) mà job sẽ kết thúc khi không tìm ra file mới. Mặc định giá trị này là 604800 giây, tức là 7 ngày
- **soft_fail**: khi job thoát ra vì timeout, nếu param này được set là True thì task sẽ có status là **skipped**, thay vì **failed**. Tức là hệ thống sẽ không xem là lỗi mà chỉ là không phát hiện file, để phân biệt với các lỗi trong quá trình dev

**Lưu ý**: Vì khi DAG thực hiện xong task processor sẽ tắt mà không tự trigger lần chạy kế tiếp (trừ khi ta lập lịch cho nó), nên ta sử dụng **TriggerDagRunOperator** để thực hiện việc trigger lại DAG này thêm một lần nữa ngay khi nó chuẩn bị kết thúc. Làm như vậy sẽ giữ cho DAG này luôn hoạt động

## Bước 3: Chạy thử nghiệm

Tạo sẵn một file csv, hoặc có thể download dữ liệu mẫu tại link: [https://github.com/datablist/sample-csv-files/blob/main/files/customers/customers-100.csv](https://github.com/datablist/sample-csv-files/blob/main/files/customers/customers-100.csv)

Bấm Trigger DAG để bắt đầu thực hiện chạy JOB. Lúc này Airflow sẽ quét thư mục SFTP Server để tìm file mới

![task1done.png](https://images2.imgbox.com/5a/d7/XDLhDTHW_o.png)

Dùng FileZilla (hoặc công cụ khác cũng được) để upload file csv lên SFTP Server tại đường dẫn thư mục đã được đặt trong DAG (`/home/files/input_data`). Lúc này ta sẽ thấy Airflow phát hiện được file mới và sẽ nhanh chóng kết thúc task **wait_for_new_file** để chuyển sang thực hiện task **process_file**

![task2done.png](https://images2.imgbox.com/cf/e0/6XdOWgrc_o.png)

Sau khi thực hiện xử lý dữ liệu xong sẽ chuyển sang task trigger để thực hiện trigger lại DAG để giữ cho DAG luôn hoạt động

![task3done.png](https://images2.imgbox.com/95/55/MLWPJnOi_o.png)

**Lưu ý**:

SFTP Sensor chỉ phát hiện có file mới thỏa điều kiện vừa được upload lên SFTP Server, chứ không trả về thông tin của file đó

## Kết luận

Như vậy bài viết này đã hướng dẫn cách sử dụng SFTP Sensor trong Airflow. Airflow còn nhiều Sensor khác hay nữa, hẹn gặp mọi người ở bài viết khác với chủ đề này nhé