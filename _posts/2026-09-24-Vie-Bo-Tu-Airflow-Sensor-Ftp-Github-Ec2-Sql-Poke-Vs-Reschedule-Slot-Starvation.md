---
title: 'Bộ tứ Airflow Sensor thực chiến: FTP, GitHub, EC2 và SQL Sensor — Cơ chế Poke vs Reschedule và giải quyết triệt để Worker Slot Starvation'
date: 2026-09-24 10:30:00 +0700
categories: [Data Engineering, Workflow Orchestration]
tags: [Apache Airflow, Airflow Sensors, Workflow Orchestration, Data Engineering, Python]
keywords: [Apache Airflow, Airflow Sensors, Workflow Orchestration, Data Engineering]
pin: false
image:
  path: /assets/img/posts/2026/bo-tu-airflow-sensor-ftp-github-ec2-sql-poke-vs-reschedule-slot-starvation/cover.webp
  alt: 'Kiến trúc cơ chế poke vs reschedule trong Airflow Sensor và giải pháp chống nghẽn worker slot starvation'
---

# I. Dẫn nhập: Đặt vấn đề và Thách thức kỹ thuật

Chắc hẳn nhiều bạn làm Data Engineering đã từng trải qua cảm giác thót tim lúc 3 giờ sáng: điện thoại rung liên hồi vì PagerDuty báo hàng loạt pipeline dữ liệu cốt lõi bị trễ SLA. Khi mình mở giao diện Apache Airflow ra xem thì đập vào mắt là một cảnh tượng hỗn loạn: cụm Celery Worker gồm 32 slots thực thi đang hoạt động ở mức 100% công suất (`capacity: 32/32`), hàng trăm task quan trọng khác đang xếp hàng dài dằng dặc ở trạng thái `queued`. Thế nhưng, khi mở Grafana kiểm tra tài nguyên CPU và RAM của máy chủ Worker thì lại thấy chúng... nhàn rỗi ở mức 1.5% đến 2%!

```
[Airflow Cluster Status: 03:15 AM]
Total Worker Slots : [████████████████████████████████] 32/32 Occupied (100%)
Worker Node CPU    : [█                                 ] 1.8% Utilization
Worker Node RAM    : [████                              ] 14.2% Utilization
Queued Task Count  : 148 tasks waiting indefinitely...
Root Cause         : 32 Sensor tasks holding slots in mode="poke"!
```

Điều gì đang xảy ra? Sau khi kiểm tra kỹ lưỡng danh sách các task đang chạy, mình phát hiện ra thủ phạm: toàn bộ 32 slots thực thi đều đang bị chiếm giữ bởi các task `Sensor`. Cụ thể là các sensor đang chờ file đối tác đẩy lên máy chủ FTP lúc nửa đêm, chờ GitHub Action build xong bản release tag, chờ máy chủ EC2 khởi động, và chờ một bảng SQL staging nạp đủ số dòng.

Vì sao một nhóm task chỉ làm nhiệm vụ "chờ đợi" lại có thể đánh sập cả một hệ sinh thái data pipeline của doanh nghiệp? Câu trả lời nằm ở chế độ hoạt động mặc định của Airflow Sensor: **`mode="poke"`**. Trong chế độ này, mỗi sensor sẽ giữ khư khư một worker thread/process của Celery hoặc Kubernetes Executor trong suốt chu kỳ chờ, liên tục gọi lệnh `time.sleep()` giữa các lần kiểm tra. Hậu quả là toàn bộ worker slot bị chiếm dụng sạch sẽ, gây ra hiện tượng **Worker Slot Starvation (Nghẽn slot worker)** và dẫn tới **Deadlock toàn cụm** — các task sinh ra dữ liệu hoặc giải phóng tài nguyên không có slot để chạy vì các sensor đang tranh giành hết chỗ đứng!

Trong bài viết này, mình sẽ cùng các bạn mổ xẻ tận gốc cơ chế vận hành bên trong của Airflow Sensor thông qua **"Bộ tứ Sensor"** thường gặp nhất trong các hệ thống dữ liệu doanh nghiệp:
1. `FTPSensor` / `SFTPSensor`: Kiểm tra file cập bến từ các đối tác ngân hàng, cổng thanh toán.
2. `GithubSensor`: Lắng nghe commit, release tag, hoặc workflow status từ kho mã nguồn.
3. `EC2InstanceStateSensor`: Giám sát trạng thái bật/tắt của các máy chủ tính toán GPU/Spot instances.
4. `SqlSensor`: Đảm bảo tính toàn vẹn dữ liệu trong các bảng staging trước khi kích hoạt transformation.

Chúng ta sẽ đi sâu vào nguyên lý `poke` vs `reschedule`, cơ chế Deferrable Operators (Triggerer), vạch trần các cạm bẫy rate limit và timeout, đồng thời xây dựng một pipeline chuẩn production giải quyết dứt điểm vấn đề cạn kiệt tài nguyên.

---

# II. Kiến trúc / Nguyên lý cốt lõi

## 1. Cơ chế nội tại của BaseSensorOperator

Về bản chất trong mã nguồn Apache Airflow, mọi sensor đều là toán tử kế thừa từ lớp `BaseSensorOperator` (thuộc module `airflow.sensors.base`). Một sensor không thực hiện biến đổi dữ liệu nặng nề mà chỉ định kỳ kiểm tra một trạng thái ngoại vi thông qua phương thức `poke(context)`:

```python
# Trích lược logic cốt lõi của BaseSensorOperator
class BaseSensorOperator(BaseOperator):
    def poke(self, context: Context) -> bool:
        """Phương thức bắt buộc override: trả về True nếu điều kiện thỏa mãn, False nếu cần chờ tiếp."""
        raise NotImplementedError()

    def execute(self, context: Context) -> Any:
        started_at = timezone.utcnow()
        while not self.poke(context):
            if (timezone.utcnow() - started_at).total_seconds() > self.timeout:
                if self.soft_fail:
                    raise AirflowSkipException("Sensor timed out; soft_fail=True -> Skipped.")
                raise AirflowSensorTimeout("Sensor timed out; failed SLA.")
            time.sleep(self.poke_interval)
        return True
```

Khi phương thức `poke()` trả về `True`, task instance kết thúc thành công (`SUCCESS`). Nhưng nếu trả về `False`, sensor sẽ phải chờ một khoảng thời gian được định nghĩa bởi `poke_interval` (mặc định là 60 giây) trước khi gọi lại `poke()`. Quá trình này lặp đi lặp lại cho đến khi đạt ngưỡng `timeout` (mặc định lên tới 604,800 giây, tương đương... 7 ngày!).

## 2. So sánh chuyên sâu: Mode `poke` vs Mode `reschedule` vs Deferrable

Điểm khác biệt chí mạng nằm ở cách toán tử xử lý khoảng thời gian nghỉ giữa các lần gọi `poke()`:

```
[Chế độ POKE: Worker Slot bị khóa chặt liên tục]
Worker Slot: [----- Poke 1 -----][ Sleep 60s ][----- Poke 2 -----][ Sleep 60s ][----- Poke 3 (True) -----> SUCCESS]
             ========================================================================================>
             Slot bị giữ 100% thời gian, Celery worker không thể nhận bất kỳ task tính toán nào khác!

[Chế độ RESCHEDULE: Worker Slot được giải phóng ngay lập tức]
Worker Slot: [ Poke 1 (False) ] -> Raise AirflowRescheduleException -> Slot giải phóng về Pool!
                    |
              [ Scheduler theo dõi thời gian poke_interval trong DB ]
                    |
Worker Slot:                   [ Poke 2 (False) ] -> Giải phóng Slot về Pool!
                                      |
                                [ Scheduler Sleep ]
                                      |
Worker Slot:                                     [ Poke 3 (True) -----> SUCCESS ]
```

### Cơ chế hoạt động của `mode="poke"`
Ở chế độ này, tiến trình worker thực thi một vòng lặp `while not self.poke(): time.sleep()`. Worker process hoàn toàn bị khóa cứng. Nếu bạn có 16 Celery worker slots và chạy 16 sensor với `mode="poke"`, toàn bộ hạ tầng Airflow của bạn sẽ rơi vào trạng thái tê liệt hoàn toàn đối với các task khác, dù CPU của máy chủ đang ở mức 0%.

### Cơ chế hoạt động của `mode="reschedule"`
Khi chuyển sang `mode="reschedule"`, nếu `poke()` trả về `False`, sensor sẽ chủ động ném ra một ngoại lệ nội bộ mang tên `AirflowRescheduleException`. Khi bắt được exception này:
1. Worker ghi nhận thời điểm tiếp theo cần kiểm tra vào bảng `task_reschedule` trong metadata database.
2. Trạng thái của task instance được chuyển thành `UP_FOR_RESCHEDULE`.
3. Tiến trình worker giải phóng hoàn toàn slot thực thi về Celery queue / Kubernetes cluster.
4. Khi đến hạn `poke_interval`, Scheduler sẽ tự động nhặt task này lên và đẩy lại vào hàng đợi để một worker rảnh rỗi thực hiện lần poke tiếp theo.

### Nâng cấp hiện đại: Deferrable Operators (Airflow 2.2+)
Bắt đầu từ phiên bản Airflow 2.2, cộng đồng giới thiệu tiến trình **Triggerer** chạy vòng lặp sự kiện bất đồng bộ Python `asyncio`. Thay vì giải phóng và tái lập lịch qua Scheduler, task chuyển quyền theo dõi cho Triggerer. Một tiến trình Triggerer duy nhất với vài trăm Megabyte RAM có thể theo dõi đồng thời hơn 10,000 sự kiện I/O mà không tiêu tốn bất kỳ worker slot nào.

## 3. Đặc thù và cạm bẫy của Bộ tứ Sensor

| Loại Sensor | Provider Package | Thách thức kỹ thuật cốt lõi | Cạm bẫy nguy hiểm nhất |
| :--- | :--- | :--- | :--- |
| **FTPSensor** | `apache-airflow-providers-ftp` | File đang được đối tác upload dở dang (Half-written file) | Đọc file rỗng hoặc file lỗi cú pháp do kiểm tra khi file chưa nạp xong |
| **GithubSensor** | `apache-airflow-providers-github` | Giới hạn hạn mức API (GitHub Rate Limit 5,000 req/h) | Quá tải request dẫn đến HTTP 403 Forbidden, làm hỏng toàn bộ pipeline CI/CD |
| **EC2InstanceStateSensor** | `apache-airflow-providers-amazon` | Trạng thái máy chủ EC2 trung gian (`pending`, `shutting-down`) | Bị AWS API throttling (`RequestLimitExceeded`) do poke quá dày đặc |
| **SqlSensor** | `apache-airflow-providers-common-sql` | Chiếm dụng connection pool trên DB đích (PostgreSQL / MySQL) | Treo kết nối PgBouncer/RDS Proxy khi giữ transaction mở trong poke mode |

Hãy phân tích sâu từng cạm bẫy:
- **Cạm bẫy FTP/SFTP Half-written file**: Khi đối tác đẩy một file 5GB lên thư mục FTP, hàm kiểm tra file tồn tại `os.path.exists()` hoặc `NLST` của FTP sẽ trả về `True` ngay khi file vừa được tạo với kích thước 0 byte. Nếu sensor lập tức báo `SUCCESS`, task ETL tiếp theo sẽ đọc phải file rác hoặc crash vì EOF bất ngờ. Giải pháp: kiểm tra kích thước file và thời gian sửa đổi (mtime) ổn định qua 2 lần poke liên tiếp.
- **Cạm bẫy GitHub Rate Limit**: GitHub REST API giới hạn mỗi Personal Access Token (PAT) chỉ được 5,000 requests/giờ. Nếu bạn cấu hình `GithubSensor` với `mode="poke"` và `poke_interval=10` cho 15 DAGs theo dõi các repo con, hệ thống sẽ thực hiện $15 \times 360 = 5,400$ requests/giờ! Chỉ sau chưa đầy 55 phút, toàn bộ token bị khóa và sensor ném lỗi HTTP 403 liên tục.

---

# III. Cài đặt / Hands-on code: Hiện thực & Tối ưu thực chiến

## 1. Cấu hình Production DAG phối hợp Bộ Tứ Sensor

Dưới đây là một DAG hoàn chỉnh chạy trong môi trường production, tích hợp đầy đủ 4 sensor với cấu hình chống nghẽn slot và xử lý ngoại lệ tối ưu:

```python
"""Production DAG orchestrating the Sensor Quartet with resilient scheduling."""
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.ftp.sensors.ftp import FTPSensor
from airflow.providers.github.sensors.github import GithubSensor
from airflow.providers.amazon.aws.sensors.ec2 import EC2InstanceStateSensor
from airflow.providers.common.sql.sensors.sql import SqlSensor
from airflow.operators.python import PythonOperator

# Default arguments áp dụng nguyên tắc phòng vệ từ xa
DEFAULT_ARGS = {
    'owner': 'data-platform',
    'depends_on_past': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=3),
    # QUY TẮC SỐNG CÒN: Luôn dùng reschedule cho mọi sensor có thời gian chờ > 2 phút
    'mode': 'reschedule',
    'poke_interval': 120,          # Kiểm tra mỗi 2 phút
    'timeout': 3600 * 3,           # Hết hạn sau 3 giờ (SLA)
    'exponential_backoff': True,   # Tự động giãn cách chu kỳ khi chờ lâu
    'max_wait': 600,               # Giãn cách tối đa 10 phút/lần
}

with DAG(
    dag_id='enterprise_sensor_quartet_orchestration',
    default_args=DEFAULT_ARGS,
    start_date=datetime(2026, 9, 20),
    schedule_interval='0 2 * * *', # 2:00 AM UTC hàng ngày
    catchup=False,
    max_active_runs=1,
    tags=['production', 'sensors', 'reschedule'],
) as dag:

    # 1. FTPSensor: Chờ file giao dịch ngân hàng cập bến thư mục đối tác
    wait_banking_csv = FTPSensor(
        task_id='wait_banking_csv',
        ftp_conn_id='partner_secure_ftp',
        path='/incoming/daily_transactions_{{ ds_nodash }}.csv',
        fail_on_transient_errors=False,
        soft_fail=False,
        pool='sensor_pool', # Gán vào pool riêng biệt
    )

    # 2. GithubSensor: Kiểm tra bản release tag mới nhất trên repository ETL
    wait_github_release = GithubSensor(
        task_id='wait_github_release_artifact',
        github_conn_id='github_enterprise_conn',
        repository_name='enterprise-org/core-data-models',
        result_processor=lambda result: bool(result and result.tag_name.startswith('v2026.')),
        poke_interval=300, # 5 phút kiểm tra 1 lần để bảo vệ API rate limit
        pool='sensor_pool',
    )

    # 3. EC2InstanceStateSensor: Giám sát máy chủ GPU tính toán nạp mô hình
    wait_gpu_ec2_running = EC2InstanceStateSensor(
        task_id='wait_gpu_ec2_running',
        aws_conn_id='aws_compute_prod',
        instance_id='i-0a8b9c1d2e3f45678',
        target_state='running',
        region_name='ap-southeast-1',
        pool='sensor_pool',
    )

    # 4. SqlSensor: Kiểm tra dữ liệu staging đã nạp đủ số lượng dòng tối thiểu
    verify_staging_records = SqlSensor(
        task_id='verify_staging_records',
        conn_id='postgres_dwh_staging',
        sql="""
            SELECT COUNT(1) >= 10000 
            FROM staging.raw_transactions 
            WHERE batch_date = '{{ ds }}';
        """,
        # Hàm kiểm tra kết quả trả về từ query scalar (phải là True)
        success=lambda record: bool(record and record[0] is True),
        failure=None,
        pool='sensor_pool',
    )

    # Task xử lý chính sau khi toàn bộ 4 điều kiện tiên quyết thỏa mãn
    def run_core_aggregation(**context):
        ti = context['ti']
        print(f"Toàn bộ điều kiện đã sẵn sàng! Bắt đầu ETL cho ngày: {context['ds']}")

    execute_core_pipeline = PythonOperator(
        task_id='execute_core_pipeline',
        python_callable=run_core_aggregation,
        pool='default_pool', # Task xử lý chạy trên pool mặc định
    )

    # Thiết lập luồng phụ thuộc: Cả 4 sensor phải sẵn sàng trước khi chạy pipeline
    [wait_banking_csv, wait_github_release, wait_gpu_ec2_running, verify_staging_records] >> execute_core_pipeline
```

## 2. Tự viết Custom Sensor với kiểm tra kích thước file ổn định (Atomic File Sensor)

Để giải quyết triệt để lỗi đọc file dở dang trên FTP hoặc File System, mình khuyến nghị các bạn xây dựng một Custom Sensor kế thừa từ `BaseSensorOperator` có khả năng theo dõi sự ổn định kích thước tệp qua 2 lần poke liên tiếp:

```python
"""Custom resilient file sensor that ensures file upload is complete before proceeding."""
from pathlib import Path
from airflow.sensors.base import BaseSensorOperator
from airflow.utils.context import Context

class ResilientStableFileSensor(BaseSensorOperator):
    """
    Sensor kiểm tra file tồn tại và kích thước không thay đổi qua 2 lần poke liên tiếp.
    Giải quyết triệt để vấn đề đối tác upload file dở dang.
    """
    template_fields = ('filepath',)

    def __init__(self, filepath: str, min_bytes: int = 1024, **kwargs):
        # Ép buộc mode='reschedule' nếu người dùng chưa chỉ định
        kwargs.setdefault('mode', 'reschedule')
        super().__init__(**kwargs)
        self.filepath = filepath
        self.min_bytes = min_bytes

    def poke(self, context: Context) -> bool:
        path = Path(self.filepath)
        ti = context['task_instance']
        
        # 1. Kiểm tra file có tồn tại trên đĩa không
        if not path.is_file():
            self.log.info(f"File {self.filepath} chưa xuất hiện. Tiếp tục chờ...")
            return False

        current_size = path.stat().st_size
        
        # 2. File phải đạt kích thước tối thiểu
        if current_size < self.min_bytes:
            self.log.info(f"File {self.filepath} tồn tại nhưng kích thước quá nhỏ ({current_size} < {self.min_bytes} bytes).")
            return False

        # 3. Lấy kích thước file ở lần poke trước từ XCom
        prev_size = ti.xcom_pull(task_ids=self.task_id, key='observed_file_size')

        if prev_size is None:
            # Lần đầu tiên nhìn thấy file, ghi nhận kích thước và yêu cầu poke lại ở chu kỳ sau
            self.log.info(f"Phát hiện file lần đầu với size={current_size} bytes. Ghi nhớ XCom để kiểm tra ở chu kỳ sau.")
            ti.xcom_push(key='observed_file_size', value=current_size)
            return False

        # 4. So sánh kích thước giữa 2 chu kỳ poke
        if current_size == prev_size:
            self.log.info(f"Xác nhận file {self.filepath} đã nạp hoàn tất! Dung lượng ổn định: {current_size} bytes.")
            return True
        else:
            self.log.warning(
                f"File đang trong quá trình ghi (Kích thước tăng từ {prev_size} lên {current_size} bytes). Tiếp tục chờ..."
            )
            ti.xcom_push(key='observed_file_size', value=current_size)
            return False
```

## 3. Bảng tra cứu cấu hình tối ưu (Sensor Production Cheat Sheet)

| Loại Sensor | Thời gian chờ ước tính | Mode khuyến nghị | `poke_interval` | `timeout` | Chiến lược bảo vệ |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **FTPSensor / SFTPSensor** | 10 phút – 4 giờ | `reschedule` | 120s – 300s | 14,400s (4h) | Kiểm tra ổn định file size; tách `ftp_pool` tối đa 4 slots |
| **GithubSensor** | 5 phút – 2 giờ | `reschedule` | 300s (5m) | 7,200s (2h) | Bắt buộc `exponential_backoff=True` tránh chạm rate limit 5k/h |
| **EC2InstanceStateSensor** | 1 phút – 15 phút | `reschedule` | 30s – 60s | 1,800s (30m) | Bắt lỗi AWS `RequestLimitExceeded`; cấp quyền IAM tối thiểu |
| **SqlSensor** | 30 giây – 30 phút | `reschedule` (nếu >2m) | 60s – 120s | 3,600s (1h) | Query gọn nhẹ qua `COUNT(1)` / `EXISTS`; không query bảng lớn |

---

# IV. Lesson learned: Tổng kết & Best Practices

Qua nhiều năm vận hành các hệ thống Airflow quy mô lớn với hàng nghìn task chạy đồng thời, mình rút ra 5 nguyên tắc sống còn giúp các bạn không bao giờ phải chịu cảnh cạn kiệt tài nguyên worker vì sensor:

1. **Tuân thủ nghiêm ngặt "Quy tắc 2 phút"**:
   - Bất kỳ sensor nào có thời gian chờ kỳ vọng lớn hơn 120 giây **BẮT BUỘC** phải được cấu hình `mode="reschedule"` hoặc chuyển đổi sang Deferrable Operator (`mode="deferrable"`).
   - Hãy đưa quy tắc này vào CI/CD pipeline bằng cách dùng linter (như `flake8-airflow` hoặc custom test) để quét mã nguồn: nếu phát hiện `mode="poke"` với `timeout > 300`, lập tức từ chối merge PR.

2. **Luôn đặt Timeout tương thích với SLA thực tế**:
   - Tuyệt đối không bao giờ để giá trị `timeout` mặc định là 7 ngày. Hãy xác định rõ ràng: "Nếu sau 2 giờ hoặc 4 giờ mà đối tác không nạp dữ liệu, pipeline có nên dừng lại để on-call can thiệp không?". Đặt timeout sát với khung giờ SLA giúp cluster tự động giải phóng tài nguyên khi xảy ra sự cố phía đối tác.

3. **Bật `exponential_backoff=True` khi gọi API bên thứ ba**:
   - Đối với các dịch vụ SaaS như GitHub, AWS API, Stripe, việc gửi request đều đặn mỗi vài giây là cách nhanh nhất để tài khoản của bạn bị đưa vào "danh sách đen" rate limit. Khi bật `exponential_backoff`, Airflow sẽ tự động nhân đôi thời gian chờ sau mỗi lần poke thất bại cho đến khi đạt `max_wait`.

4. **Sử dụng `soft_fail=True` cho các nguồn dữ liệu thứ cấp**:
   - Nếu dữ liệu từ một sensor không bắt buộc cho toàn bộ báo cáo cuối ngày (ví dụ: file tỉ giá tham khảo bổ sung), hãy thiết lập `soft_fail=True`. Khi hết hạn timeout, task sẽ chuyển sang trạng thái `SKIPPED` thay vì `FAILED`, giúp các task hạ nguồn tiếp tục thực thi bình thường mà không làm tắc nghẽn luồng dữ liệu.

5. **Cách ly Sensor vào Airflow Pool riêng (`sensor_pool`)**:
   - Trong giao diện Airflow (Admin -> Pools), hãy tạo một pool riêng tên là `sensor_pool` với số lượng slots giới hạn (ví dụ: 10% đến 15% tổng số slots của worker cluster). Bằng cách gán `pool='sensor_pool'` cho tất cả các sensor, bạn đảm bảo rằng dù có hàng trăm sensor cùng hoạt động một lúc, chúng cũng không bao giờ có thể chiếm dụng hết các slots dành cho task tính toán nặng.
