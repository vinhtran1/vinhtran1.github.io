---
title: 'Thiết lập cảnh báo Airflow thời gian thực qua Slack Webhook: Giao diện Block Kit tương tác và on_failure_callback chuyên sâu'
date: 2026-09-24 12:30:00 +0700
categories: [DevOps, Monitoring]
tags: [Apache Airflow, Slack Webhook, Alerting, DevOps, Monitoring]
keywords: [Apache Airflow, Slack Webhook, Alerting, DevOps]
pin: false
image:
  path: /assets/img/posts/2026/thiet-lap-canh-bao-airflow-qua-slack-webhook-block-kit-va-on-failure-callback/cover.webp
  alt: 'Kiến trúc hệ thống cảnh báo Airflow thời gian thực qua Slack Webhook với giao diện Block Kit tương tác'
---

# I. Dẫn nhập: Đặt vấn đề và Thách thức kỹ thuật

Hãy tưởng tượng một tình huống quen thuộc mà bất kỳ đội ngũ kỹ thuật dữ liệu nào cũng từng chạm trán: Đúng 7 giờ 50 sáng, khi bạn đang chuẩn bị pha cà phê thì nhận được tin nhắn trực tiếp từ Giám đốc Tài chính: *"Dashboard doanh thu hôm nay vẫn trắng tinh, 8h30 có cuộc họp với Ban điều hành, team kiểm tra gấp!"*. Cả đội vội vã mở máy tính, bật VPN, đăng nhập vào Airflow Webserver thì phát hiện ra pipeline ETL tổng hợp doanh thu đã bị lỗi từ... 4 giờ 15 sáng! 

Vậy trong suốt gần 4 tiếng đồng hồ đó, hệ thống giám sát ở đâu? Khi mở hòm thư điện tử, mình thấy một email cảnh báo thất bại nằm im lìm trong mục Spam (Junk Folder). 

```
[Default Email Alert - Trapped in Spam]
From: airflow-alert@internal-domain.com
Subject: Airflow alert: <TaskInstance: dwh_daily_sales.aggregate_revenue 2026-09-24T04:00:00+00:00 [failed]>
Body: Try 3 of 3. Exception: Connection refused to Postgres DB. 
(No deep link, no quick action button, zero immediate visibility)
```

Đây chính là cơn ác mộng kinh điển của việc giám sát pipeline theo cách truyền thống:
1. **`email_on_failure=True`**: Hầu hết email cảnh báo đều bị trôi vào hộp thư rác hoặc bị kỹ sư thiết lập bộ lọc tự động chuyển tiếp vì quá nhiều tin nhắn rác. Email không cung cấp thông tin ngữ cảnh trực quan, không có đường link trực tiếp tới đúng dòng log lỗi.
2. **Cảnh báo kiểu "Task ở cuối DAG"**: Rất nhiều bạn áp dụng giải pháp đặt một task `SlackWebhookOperator` ở cuối workflow. Đây là một cạm bẫy thiết kế nghiêm trọng: Nếu một task ở giữa đường ống bị crash, toàn bộ luồng thực thi dừng lại ngay lập tức và task gửi thông báo ở cuối sẽ **không bao giờ được chạy**!
3. **Tin nhắn Slack thô sơ dạng Plain-Text**: "Task X bị fail". Khi nhận được tin này, kỹ sư trực ca (on-call) phải mất 5–10 phút mở giao diện Airflow, tìm đúng DAG, tìm đúng Run ID, bấm vào Task Instance, mở tab Logs và cuộn qua hàng nghìn dòng văn bản rối rắm để tìm ra nguyên nhân.

Giải pháp chuẩn mực cho môi trường Enterprise là xây dựng một hệ sinh thái cảnh báo thời gian thực thông qua cơ chế **Native Callback (`on_failure_callback`)** của Apache Airflow kết hợp với ngôn ngữ thiết kế giao diện tương tác **Slack Block Kit UI**. 

Trong bài viết này, mình sẽ cùng các bạn bóc tách cơ chế vòng đời callback của Airflow, phân tích từ điển ngữ cảnh (`context`), xây dựng module gửi cảnh báo với giao diện thẻ Block Kit đẹp mắt, tự động trích xuất đoạn lỗi traceback cốt lõi, cung cấp nút bấm deep-link vào thẳng dòng log trong Webserver, và mở rộng tính năng chạy lại task chỉ với một cú nhấp chuột (One-click remediation).

---

# II. Kiến trúc / Nguyên lý cốt lõi

## 1. Vòng đời Task và Cơ chế Callback trong Apache Airflow

Để hiểu cách hoạt động của callback, chúng ta cần nhìn lại máy trạng thái hữu hạn (Finite State Machine) của một `TaskInstance`:

```
               +-------------------------------------------------------+
               |                        QUEUED                         |
               +-------------------------------------------------------+
                                           |
                                           v
               +-------------------------------------------------------+
               |                        RUNNING                        |
               +-------------------------------------------------------+
                         |                                   |
                         | (Success)                         | (Exception caught)
                         v                                   v
        +---------------------------------+        +--------------------+
        |             SUCCESS             |        |  Retry Remaining?  |
        +---------------------------------+        +--------------------+
                         |                               /        \
                         v                              / (Yes)    \ (No)
              [ on_success_callback ]                  v            v
                                            +--------------+  +--------------+
                                            | UP_FOR_RETRY |  |    FAILED    |
                                            +--------------+  +--------------+
                                                   |                 |
                                                   v                 v
                                         [ on_retry_callback ] [ on_failure_callback ]
```

### Phân cấp các loại Callback:
1. **Task-level Callbacks**:
   - `on_failure_callback`: Kích hoạt khi task instance chuyển sang trạng thái `FAILED` (sau khi đã dùng hết toàn bộ số lần `retries`).
   - `on_retry_callback`: Kích hoạt mỗi khi một lần thử thất bại nhưng vẫn còn lượt retry (`UP_FOR_RETRY`).
   - `on_success_callback`: Kích hoạt khi task hoàn thành mỹ mãn (`SUCCESS`).
2. **DAG-level Callbacks**:
   - `dag.on_failure_callback`: Kích hoạt khi toàn bộ DAG Run bị đánh dấu thất bại.
   - `dag.on_success_callback`: Kích hoạt khi toàn bộ các task nhánh kết thúc thành công.
   - `sla_miss_callback`: Kích hoạt khi thời gian chạy vượt quá ngưỡng cam kết SLA.

### Khám phá Từ điển Ngữ cảnh (`context` dictionary)
Khi callback được kích hoạt, Airflow sẽ truyền vào một từ điển chứa toàn bộ siêu dữ liệu của lần chạy đó. Các trường thông tin quý giá nhất bao gồm:
- `task_instance` (hoặc `ti`): Đối tượng `TaskInstance` chứa thông tin về `try_number`, `max_tries`, `duration`, `hostname`, `pool`.
- `exception`: Chứa đối tượng ngoại lệ Python nguyên bản đã làm sập task (không cần phải đọc file log trên đĩa!).
- `execution_date` / `logical_date`: Mốc thời gian logic của chu kỳ dữ liệu.
- `dag`: Đối tượng DAG hiện tại.
- `ti.log_url`: **Đường link URL trỏ thẳng tới tab xem log của task trên giao diện Airflow Webserver**.

## 2. Ngôn ngữ thiết kế giao diện Slack Block Kit

Thay vì gửi một chuỗi văn bản không định dạng, **Slack Block Kit** là một framework UI dựa trên JSON cho phép chúng ta dựng các thẻ tương tác (Cards) chuyên nghiệp:

```
+-----------------------------------------------------------------------------------+
| 🔴 [ALERT] Task Failure: dwh_daily_sales.aggregate_revenue                        |
+-----------------------------------------------------------------------------------+
| Môi trường: Production                   | Thời gian logic: 2026-09-24 04:00 UTC  |
| Lần thử   : 3 / 3 (Hết lượt retry)       | Thời lượng chạy: 142.6 giây            |
+-----------------------------------------------------------------------------------+
| 💥 Nguyên nhân lỗi (Exception Traceback):                                         |
| ```                                                                               |
| psycopg2.OperationalError: could not connect to server: Connection refused        |
| Is the server running on host "db.internal.lan" and accepting connections?        |
| ```                                                                               |
+-----------------------------------------------------------------------------------+
| [ 🔍 Xem Log Chi Tiết ]       [ 📊 Grid View ]       [ ⚡ Rerun Task (Admin) ]     |
+-----------------------------------------------------------------------------------+
```

### Cấu trúc JSON chuẩn của Block Kit Card:
- **Header Block**: Tiêu đề in đậm, icon trạng thái màu đỏ (`#E01E5A`) biểu thị sự cố nghiêm trọng.
- **Section Block (Fields)**: Chia layout thành 2 cột cân xứng để hiển thị DAG ID, Task ID, Hostname worker, Run ID, và số lần thử lại.
- **Code Block**: Chứa đoạn lỗi Exception đã được cắt tỉa gọn gàng. Slack giới hạn một text block tối đa 3,000 ký tự; vì vậy chúng ta cần trích xuất 10–15 dòng cuối của traceback và giới hạn độ dài < 2,500 ký tự để không bị lỗi `invalid_blocks`.
- **Actions Block**: Chứa các nút bấm (`elements: [{"type": "button", ...}]`) tích hợp đường dẫn deep-link.

## 3. Nguyên tắc an toàn khi viết Callback (Fail-Safe Callback)

Một sai lầm rất nguy hiểm mà nhiều kỹ sư mắc phải là viết hàm callback thiếu cơ chế bảo vệ. Hãy nhớ rằng: **Mã callback được thực thi ngay trong tiến trình của Airflow Worker hoặc Airflow Scheduler!**

Nếu trong hàm callback:
- Bạn gọi một API Slack bên ngoài mà không đặt `timeout` kết nối.
- Slack API gặp sự cố hoặc trả về mã lỗi HTTP 500 / 429.
- Hàm callback ném ra một ngoại lệ chưa được bắt (`unhandled exception`).

Hậu quả là tiến trình Scheduler hoặc TaskRunner có thể bị crash, task instance rơi vào trạng thái lơ lửng không thể cập nhật DB, và hệ thống tự động đánh dấu nhầm thành **Zombie Task**. Do đó, quy tắc số một: **Toàn bộ nội dung callback phải được bọc trong khối `try...except Exception` an toàn và thiết lập timeout mạng tối đa 5 đến 10 giây.**

---

# III. Cài đặt / Hands-on code: Hiện thực & Tối ưu thực chiến

## 1. Xây dựng Module Alert chuẩn `airflow_slack_notifier.py`

Dưới đây là module hoàn chỉnh phục vụ việc tạo thẻ Block Kit và gửi thông báo qua Slack Incoming Webhook:

```python
"""Enterprise Slack Alerting Module for Apache Airflow using Block Kit."""
import json
import logging
import traceback
from typing import Dict, Any
import requests
from airflow.models import Variable

logger = logging.getLogger("airflow.slack_notifier")

# Mặc định timeout cho request gọi sang Slack
SLACK_REQUEST_TIMEOUT_SECONDS = 8
MAX_TRACEBACK_LENGTH = 2000

def extract_clean_traceback(context: Dict[str, Any]) -> str:
    """Trích xuất đoạn traceback gọn gàng từ context, giới hạn độ dài an toàn."""
    exception = context.get('exception')
    if exception is not None:
        if isinstance(exception, Exception):
            tb = "".join(traceback.format_exception(type(exception), exception, exception.__traceback__))
            # Lấy 15 dòng cuối cùng của traceback
            lines = tb.strip().splitlines()
            clean_tb = "\n".join(lines[-15:])
            return clean_tb[:MAX_TRACEBACK_LENGTH]
        return str(exception)[:MAX_TRACEBACK_LENGTH]
    return "Không có chi tiết Exception. Vui lòng kiểm tra log trên Airflow UI."

def build_slack_block_kit_payload(context: Dict[str, Any]) -> Dict[str, Any]:
    """Xây dựng cấu trúc Block Kit JSON tương tác chuẩn Enterprise."""
    ti = context.get('task_instance')
    dag = context.get('dag')
    logical_date = context.get('logical_date') or context.get('execution_date')
    
    dag_id = dag.dag_id if dag else "Unknown_DAG"
    task_id = ti.task_id if ti else "Unknown_Task"
    run_id = context.get('run_id', 'N/A')
    try_number = ti.try_number if ti else 1
    max_tries = (ti.max_tries + 1) if ti else 1
    duration = f"{ti.duration:.1f}s" if (ti and ti.duration) else "N/A"
    log_url = ti.log_url if ti else "http://localhost:8080"
    
    error_snippet = extract_clean_traceback(context)
    
    payload = {
        "text": f"🚨 [Airflow Alert] Sự cố Task: {dag_id}.{task_id}",
        "attachments": [
            {
                "color": "#E01E5A", # Màu đỏ cảnh báo khẩn cấp
                "blocks": [
                    {
                        "type": "header",
                        "text": {
                            "type": "plain_text",
                            "text": f"🚨 Airflow Task Failure: {task_id}",
                            "emoji": True
                        }
                    },
                    {
                        "type": "section",
                        "fields": [
                            {"type": "mrkdwn", "text": f"*Workflow (DAG):*\n`{dag_id}`"},
                            {"type": "mrkdwn", "text": f"*Môi trường:*\n`Production`"},
                            {"type": "mrkdwn", "text": f"*Logical Date:*\n{logical_date.strftime('%Y-%m-%d %H:%M:%S UTC') if logical_date else 'N/A'}"},
                            {"type": "mrkdwn", "text": f"*Lần thử lại:*\n`{try_number} / {max_tries}`"},
                            {"type": "mrkdwn", "text": f"*Thời gian chạy:*\n`{duration}`"},
                            {"type": "mrkdwn", "text": f"*Run ID:*\n`{run_id[:28]}...`"}
                        ]
                    },
                    {
                        "type": "divider"
                    },
                    {
                        "type": "section",
                        "text": {
                            "type": "mrkdwn",
                            "text": f"*💥 Chi tiết lỗi (Exception Traceback):*\n```{error_snippet}```"
                        }
                    },
                    {
                        "type": "actions",
                        "elements": [
                            {
                                "type": "button",
                                "text": {
                                    "type": "plain_text",
                                    "text": "🔍 Xem Log Chi Tiết",
                                    "emoji": True
                                },
                                "url": log_url,
                                "style": "primary"
                            },
                            {
                                "type": "button",
                                "text": {
                                    "type": "plain_text",
                                    "text": "📊 Grid View",
                                    "emoji": True
                                },
                                "url": f"{log_url.split('/task?')[0]}/grid?dag_id={dag_id}"
                            }
                        ]
                    }
                ]
            }
        ]
    }
    return payload

def slack_failure_callback(context: Dict[str, Any]) -> None:
    """
    Hàm Callback gửi thông báo sang Slack khi task thất bại.
    Được thiết kế theo nguyên tắc Fail-Safe (không bao giờ làm crash tiến trình Airflow).
    """
    try:
        # Lấy Webhook URL từ Airflow Variables hoặc Environment
        webhook_url = Variable.get("SLACK_ALERT_WEBHOOK_URL", default_var=None)
        if not webhook_url:
            logger.warning("Biến SLACK_ALERT_WEBHOOK_URL chưa được cấu hình. Bỏ qua gửi Slack alert.")
            return

        payload = build_slack_block_kit_payload(context)
        response = requests.post(
            webhook_url,
            data=json.dumps(payload),
            headers={"Content-Type": "application/json"},
            timeout=SLACK_REQUEST_TIMEOUT_SECONDS
        )

        if response.status_code != 200:
            logger.error(f"Slack API phản hồi lỗi HTTP {response.status_code}: {response.text}")
        else:
            logger.info("Đã gửi thành công cảnh báo Block Kit tới Slack.")

    except Exception as e:
        # Nuốt lỗi an toàn để bảo vệ Scheduler/Worker process
        logger.exception(f"Lỗi ngoài dự kiến khi thực thi slack_failure_callback: {str(e)}")
```

## 2. Triển khai Nút Bấm Deep-Link chính xác vào Webserver

Một điểm các bạn cần đặc biệt lưu ý: Thuộc tính `ti.log_url` trong Airflow được tự động lắp ráp dựa trên cấu hình `base_url` trong tệp `airflow.cfg`. Nếu bạn không định cấu hình trường này, đường dẫn sinh ra sẽ có dạng `http://localhost:8080/task?dag_id=...` — và khi kỹ sư on-call bấm vào trên điện thoại hoặc máy tính cá nhân, trình duyệt sẽ báo lỗi `Connection Refused`!

Hãy đảm bảo cấu hình `airflow.cfg` hoặc biến môi trường container:
```ini
[webserver]
# Cấu hình Domain hoặc Ingress chính xác của Airflow Webserver
base_url = https://airflow.internal-domain.com
```

## 3. Cấu hình Kế thừa Toàn cục (Global Default Args)

Thay vì phải gán callback cho từng task đơn lẻ, các bạn có thể nhúng trực tiếp hàm callback này vào `default_args` của DAG. Như vậy, 100% các task trong DAG sẽ tự động được bảo vệ:

```python
"""Sample Production DAG using global Slack failure callback."""
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from airflow_slack_notifier import slack_failure_callback

DEFAULT_ARGS = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=3),
    # Tự động kích hoạt khi task dùng hết 2 lần retry mà vẫn thất bại
    'on_failure_callback': slack_failure_callback,
}

with DAG(
    dag_id='finance_daily_aggregation_pipeline',
    default_args=DEFAULT_ARGS,
    start_date=datetime(2026, 9, 20),
    schedule_interval='15 4 * * *',
    catchup=False,
    max_active_runs=1,
    tags=['finance', 'critical', 'sla_30m'],
) as dag:

    extract_bank_records = BashOperator(
        task_id='extract_bank_records',
        bash_command='echo "Đang trích xuất dữ liệu ngân hàng..." && sleep 5',
    )

    def process_financial_ledger():
        # Giả lập lỗi kết nối database để kiểm thử cảnh báo Block Kit
        raise ConnectionError("Không thể kết nối tới Production Ledger DB sau 3 lần thử.")

    aggregate_ledger = PythonOperator(
        task_id='aggregate_financial_ledger',
        python_callable=process_financial_ledger,
    )

    extract_bank_records >> aggregate_ledger
```

## 4. Nâng cao: Tích hợp Interactive Button "Clear Task" qua Slack Interactivity

Trong môi trường DevOps hiện đại, bạn có thể biến thông báo Slack thành một bảng điều khiển tương tác. Khi kỹ sư nhận được cảnh báo, thay vì phải mở laptop, họ có thể bấm ngay nút **⚡ Rerun Task** trên giao diện Slack điện thoại:

```
[Kỹ sư bấm nút 'Rerun Task' trên Slack]
                     |
                     v
       (Slack Interactivity Webhook)
                     |
                     v
         [AWS API Gateway / Lambda]
                     |
                     v
       (Airflow REST API: Basic Auth / Bearer)
  POST /api/v1/dags/{dag_id}/clearTaskInstances
                     |
                     v
   [Task Instance chuyển trạng thái sang NULL]
                     |
                     v
[Scheduler tự động lập lịch chạy lại tức thì!]
```

Dưới đây là đoạn mã Python ngắn chạy trên AWS Lambda nhận Webhook tương tác từ Slack và kích hoạt Airflow REST API:

```python
"""AWS Lambda function handling Slack Interactive Payload to clear Airflow Task."""
import json
import os
import requests

AIRFLOW_BASE_URL = os.environ.get("AIRFLOW_BASE_URL", "https://airflow.internal-domain.com")
AIRFLOW_USER = os.environ.get("AIRFLOW_API_USER")
AIRFLOW_PASS = os.environ.get("AIRFLOW_API_PASS")

def lambda_handler(event, context):
    body = event.get("body", "")
    # Parse payload từ Slack Interactivity
    from urllib.parse import parse_qs
    parsed_body = parse_qs(body)
    payload = json.loads(parsed_body['payload'][0])
    
    action_value = payload['actions'][0]['value']  # Chứa "dag_id:task_id:execution_date"
    dag_id, task_id, execution_date = action_value.split(":")
    
    # Gọi Airflow 2.x REST API để clear task instance
    clear_endpoint = f"{AIRFLOW_BASE_URL}/api/v1/dags/{dag_id}/clearTaskInstances"
    req_body = {
        "dry_run": False,
        "task_ids": [task_id],
        "start_date": execution_date,
        "end_date": execution_date,
        "only_failed": True,
        "include_subdags": False,
        "reset_dag_runs": True
    }
    
    response = requests.post(
        clear_endpoint,
        json=req_body,
        auth=(AIRFLOW_USER, AIRFLOW_PASS),
        headers={"Content-Type": "application/json"},
        timeout=10
    )
    
    return {
        "statusCode": 200,
        "body": json.dumps({"text": f"✅ Đã yêu cầu Airflow chạy lại task `{task_id}`!"})
    }
```

---

# IV. Lesson learned: Tổng kết & Best Practices

Hệ thống cảnh báo tốt là hệ thống đem lại sự an tâm chứ không phải sự hoảng loạn. Dưới đây là 5 kinh nghiệm thực tế khi thiết lập cảnh báo Slack cho Airflow mà mình đúc kết được:

1. **Chỉ bắn cảnh báo khẩn khi HẾT LƯỢT RETRY**:
   - Nếu một task được cấu hình `retries=3`, việc nó bị lỗi ở lần 1 do mạng chập chờn là điều hoàn toàn bình thường. Đừng gửi tin nhắn báo động đỏ vào kênh chung ở lần 1, vì điều này gây ra hiện tượng **Alert Fatigue (Nhờn cảnh báo)**.
   - Hãy sử dụng `on_retry_callback` cho một kênh debug riêng biệt (`#airflow-retries`), và chỉ kích hoạt `on_failure_callback` gửi vào kênh chính khi task thực sự gục ngã sau lần thử cuối cùng.

2. **Phân cấp kênh Slack theo mức độ nghiêm trọng (Alert Routing)**:
   - Các pipeline lõi (doanh thu, thanh toán) gửi cảnh báo vào kênh `#alerts-critical` kèm tag trực tiếp nhóm on-call (`<!subteam^ID_DEV_LEAD>`).
   - Các pipeline thu thập dữ liệu thứ cấp gửi vào kênh `#alerts-warning` không kèm mention để tránh làm phiền giấc ngủ của kỹ sư lúc nửa đêm.

3. **Chống "Bão tin nhắn" bằng DAG-level Callback**:
   - Khi một database nguồn bị sập, một DAG có 60 task song song sẽ đồng loạt fail, làm bắn 60 tin nhắn Slack liên tiếp trong 3 giây.
   - Để tránh làm tê liệt kênh chat, đối với các DAG dạng Fan-out lớn, các bạn nên ưu tiên sử dụng `dag.on_failure_callback` để tổng hợp thành **MỘT** tin nhắn duy nhất tóm tắt toàn bộ các task bị ảnh hưởng trong DAG Run đó.

4. **Luôn cắt tỉa Exception Traceback**:
   - Slack có giới hạn nghiêm ngặt 3,000 ký tự cho mỗi text block. Nếu bạn in toàn bộ traceback của PySpark hay Java stack trace, request sẽ bị Slack từ chối với mã lỗi `HTTP 400 invalid_blocks`. Hãy luôn giới hạn độ dài của chuỗi lỗi trong khoảng 1,500 đến 2,000 ký tự.

5. **Bảo mật tuyệt đối Webhook URL**:
   - Tuyệt đối không hardcode URL Webhook của Slack trực tiếp trong mã nguồn DAG. Hãy lưu trữ nó trong Airflow Variables, Airflow Connections, hoặc dịch vụ quản lý bí mật như AWS Secrets Manager.
