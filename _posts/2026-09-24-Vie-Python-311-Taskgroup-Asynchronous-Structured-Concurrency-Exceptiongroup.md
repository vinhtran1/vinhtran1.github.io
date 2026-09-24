---
title: 'Python 3.11 TaskGroup & Structured Concurrency: Quản lý bất đồng bộ hiện đại và xử lý ngoại lệ đa điểm với ExceptionGroup'
date: 2026-09-24 07:00:00 +0700
categories: [Python, Concurrency]
tags: [Python, Asyncio, Concurrency, ExceptionGroup, Performance]
keywords: [Python, Asyncio, Concurrency, ExceptionGroup]
pin: false
image:
  path: /assets/img/posts/2026/python-311-taskgroup-asynchronous-structured-concurrency-exceptiongroup/cover.webp
  alt: 'Kiến trúc Structured Concurrency với asyncio.TaskGroup và xử lý ngoại lệ đa luồng ExceptionGroup trong Python 3.11'
---

Chào các bạn, khi làm việc với lập trình bất đồng bộ (`asyncio`) trong các phiên bản Python trước 3.11, chắc hẳn nhiều bạn đã từng trải qua cảm giác bất lực khi đối mặt với những lỗi "chạy ngầm bí ẩn" trên production. 

Hãy tưởng tượng một kịch bản quen thuộc: một API endpoint của các bạn gọi đồng thời 5 tác vụ con bằng `asyncio.gather()`. Tác vụ số 1 bị lỗi `DatabaseTimeout`, tác vụ số 2 bị `ConnectionRefusedError`, trong khi 3 tác vụ còn lại... tiếp tục chạy ngầm trong hư vô mà không có ai quản lý hay dọn dẹp. Đây chính là hiện tượng **Coroutine mồ côi (Orphaned / Leaked Tasks)** — nguồn cơn gây rò rỉ socket, cạn kiệt connection pool và làm treo hệ thống trong âm thầm.

Trong bài viết hôm nay, mình sẽ cùng các bạn khám phá bước chuyển mình mang tính lịch sử của Python 3.11: **Structured Concurrency (Đồng quy có cấu trúc)** với `asyncio.TaskGroup`, đi kèm cơ chế gom nhóm và xử lý ngoại lệ đa luồng mang tính cách mạng thông qua `ExceptionGroup` và cú pháp `except*`.

---

# I. Dẫn nhập: Lý thuyết & Nguyên lý nền tảng

### 1. Nguồn gốc của Structured Concurrency
Khái niệm **Structured Concurrency** được khởi xướng bởi Martin Sústrik (tác giả ZeroMQ) và được hoàn thiện về mặt lý thuyết bởi Nathaniel J. Smith (tác giả thư viện Trio). Ý tưởng cốt lõi của nó bắt nguồn sâu xa từ bài luận kinh điển năm 1968 của Edsger Dijkstra: *"Go To Statement Considered Harmful"*.

Trong kỷ nguyên lập trình tuần tự trước đây, lệnh `goto` cho phép con trỏ chương trình nhảy tùy tiện tới bất kỳ vị trí nào trong mã nguồn, khiến luồng thực thi trở thành một "mớ bòng bong" (spaghetti code) không thể kiểm soát. Cuộc cách mạng lập trình có cấu trúc (Structured Programming) đã thay thế `goto` bằng các khối lệnh có phạm vi rõ ràng: `if/else`, `while`, `for`, và các hàm (functions). Điểm chung của chúng là: **có một điểm vào xác định và bắt buộc phải có một điểm ra thống nhất**.

Thế nhưng, khi bước sang kỷ nguyên lập trình song song và bất đồng bộ, chúng ta lại vô tình tạo ra một dạng `goto` mới: **Fire-and-forget Tasks**. Khi các bạn gọi `asyncio.create_task()` mà không có một cơ chế bao bọc nghiêm ngặt, task đó sẽ trôi dạt tự do trên event loop, hoàn toàn tách rời khỏi phạm vi từ vựng (lexical scope) của hàm đã sinh ra nó.

```
Mô hình Unstructured (Vô tổ chức):
[Caller Scope] ────(create_task)────> [Task A trôi nổi ngoài scope]
       │                                     │ (Gặp lỗi crash hoặc treo vô hạn)
       ▼ (Scope đã return / đóng socket)     ▼
[Resource Leak: Coroutine mồ côi vẫn chạy ngầm!]
```

**Nguyên tắc vàng của Structured Concurrency**:
> Mọi luồng tính toán đồng quy (concurrent tasks) khi được phân nhánh (fork) bắt buộc phải có điểm hội tụ (join) nằm trọn vẹn trong cùng một cấu trúc phạm vi từ vựng bao bọc nó. Một scope cha không bao giờ được phép kết thúc chừng nào các task con của nó chưa kết thúc hoặc chưa được dọn dẹp sạch sẽ.

### 2. Cuộc cách mạng trong Python 3.11: PEP 654 và PEP 678
Để hiện thực hóa triết lý này vào CPython core, Python 3.11 đã giới thiệu hai đề xuất cải tiến lớn:
- **PEP 654 (Exception Groups and except\*)**: Định nghĩa hai kiểu ngoại lệ mới là `ExceptionGroup` và `BaseExceptionGroup`. Khi nhiều task đồng thời ném ra lỗi độc lập, Python không còn phải "chọn bừa" một lỗi đầu tiên để re-raise và nuốt các lỗi còn lại nữa; thay vào đó, toàn bộ các lỗi sẽ được đóng gói thành một cây phân cấp lỗi hoàn chỉnh.
- **PEP 678 (Enriching Exceptions with Notes)**: Cung cấp phương thức `exc.add_note("context information")` giúp các bạn bổ sung ngữ cảnh chẩn đoán vào từng exception nhánh mà không làm biến dạng traceback gốc.

---

# II. Kiến trúc & So sánh thực tế

### 1. Kiến trúc luồng vận hành của `asyncio.TaskGroup`
Khác với `asyncio.gather()`, `asyncio.TaskGroup` được thiết kế hoạt động như một Asynchronous Context Manager thông qua cú pháp `async with asyncio.TaskGroup() as tg:`.

Khi một task con bất kỳ bên trong group gặp ngoại lệ không được xử lý:
1. `TaskGroup` lập tức kích hoạt cơ chế **Cascade Cancellation**: gọi `task.cancel()` đối với tất cả các task con còn lại đang chạy trong group.
2. Khối `__aexit__` của Context Manager sẽ kiên nhẫn đợi (await) toàn bộ các task con hoàn thành chu trình xử lý `CancelledError` và giải phóng tài nguyên.
3. Toàn bộ các ngoại lệ phát sinh trong quá trình thực thi sẽ được gom vào một `ExceptionGroup` duy nhất và re-raise ra bên ngoài context.

```
Sơ đồ Cancellation Cascading & Exception Grouping trong TaskGroup:

[async with asyncio.TaskGroup() as tg]
      │
      ├──> Fork Task 1 (Kiểm tra kho)    ──[Đang chạy...]──> Bị huỷ bởi TG ──[CancelledError handled]
      ├──> Fork Task 2 (Trừ thẻ tín dụng)──[THẤT BẠI: HTTP 504 Timeout!] ──┐
      └──> Fork Task 3 (Gọi đơn vị ship)  ──[THẤT BẠI: ConnectionRefused!] ─┴─┐
                                                                              │
[TaskGroup Context Exit: Join Point] <────────────────────────────────────────┘
      │
      └──> Re-raise ExceptionGroup("unhandled errors in a TaskGroup", [TimeoutError, ConnectionRefusedError])
            │
            ├──> except* TimeoutError: Ghi log cảnh báo cổng thanh toán nghẽn
            └──> except* ConnectionRefusedError: Gửi alert khẩn cấp hạ tầng logistics sập
```

### 2. So sánh đa chiều: `gather` vs `create_task` vs `TaskGroup`

| Tiêu chí so sánh | `asyncio.gather()` | `asyncio.create_task()` đơn lẻ | `asyncio.TaskGroup()` (Python 3.11+) |
| :--- | :--- | :--- | :--- |
| **Ranh giới phạm vi (Scope)** | Cục bộ theo biểu thức `gather` | Toàn cục / Unbounded (bắn rồi quên) | Giới hạn tuyệt đối trong `async with` |
| **Xử lý Task mồ côi (Orphan leaks)** | Dễ bị rò rỉ nếu một task fail | Cực kỳ nguy hiểm nếu quên quản lý reference | **Triệt tiêu 100%**: tự động cancel task anh em |
| **Xử lý đa ngoại lệ đồng thời** | Chỉ re-raise lỗi đầu tiên (nuốt lỗi sau) | Phải tự `try/except` thủ công từng task | **Hỗ trợ tự nhiên** qua `ExceptionGroup` & `except*` |
| **Bẫy nuốt lỗi vô tình** | Rất cao khi đặt `return_exceptions=True` | Cao nếu quên `task.exception()` | **Không thể nuốt lỗi vô ý** |
| **Đồng bộ ContextVar & Tracing** | Dễ mất ngữ cảnh nếu không cẩn thận | Cần truyền context thủ công | Tự động lan truyền ContextVar an toàn |
| **Khả năng dọn dẹp tài nguyên** | Phụ thuộc vào lập trình viên bắt lỗi | Rời rạc, dễ sót | Tự động join và await cancellation trước khi thoát |

---

# III. Cài đặt & Thực hành code: Triển khai thực chiến

Để các bạn thấy rõ sức mạnh thực chiến của `TaskGroup` và `except*`, chúng ta hãy cùng xây dựng một mô-đun Checkout đơn hàng E-Commerce. Trong quy trình checkout, hệ thống cần gọi 3 dịch vụ vi mô (microservices) bất đồng bộ:
1. `check_inventory`: Kiểm tra tồn kho hàng hóa (tác vụ kéo dài 0.8s).
2. `charge_credit_card`: Xử lý thanh toán thẻ tín dụng (bị lỗi Gateway Timeout sau 0.4s).
3. `reserve_shipping`: Giữ chỗ đơn vị vận chuyển (bị từ chối kết nối `ConnectionRefusedError` sau 0.3s).

Dưới đây là mã nguồn hoàn chỉnh, tự chứa (self-contained) và có thể chạy trực tiếp trên Python 3.11+:

```python
import asyncio
import time
from typing import Dict, Any

class PaymentGatewayTimeout(Exception):
    """Ngoại lệ ném ra khi cổng thanh toán trực tuyến phản hồi quá thời gian quy định."""
    pass

class ShippingServiceUnavailable(Exception):
    """Ngoại lệ ném ra khi máy chủ đối tác vận chuyển không thể kết nối."""
    pass

async def check_inventory(order_id: str) -> Dict[str, Any]:
    """Tác vụ 1: Kiểm tra tồn kho hàng hóa."""
    print(f"[{time.strftime('%X')}] [Inventory] Đang kiểm tra tồn kho cho đơn {order_id}...")
    try:
        # Giả lập tác vụ kiểm kho mất 0.8 giây
        await asyncio.sleep(0.8)
        print(f"[{time.strftime('%X')}] [Inventory] Thành công! Đủ hàng trong kho.")
        return {"status": "in_stock", "available_units": 15}
    except asyncio.CancelledError:
        print(f"[{time.strftime('%X')}] [Inventory] Task bị cancel do các dịch vụ khác gặp sự cố! Dọn dẹp lock kho an toàn.")
        # Luôn luôn re-raise CancelledError trong asyncio
        raise

async def charge_credit_card(order_id: str, amount: float) -> Dict[str, Any]:
    """Tác vụ 2: Xử lý trừ tiền thẻ tín dụng khách hàng."""
    print(f"[{time.strftime('%X')}] [Payment] Đang trừ tiền đơn {order_id} số tiền {amount:,.0f} VND...")
    await asyncio.sleep(0.4)
    print(f"[{time.strftime('%X')}] [Payment] LỖI: Cổng thanh toán bị Timeout!")
    raise PaymentGatewayTimeout(f"Gateway không phản hồi sau 400ms đối với giao dịch {order_id}")

async def reserve_shipping(order_id: str, delivery_address: str) -> Dict[str, Any]:
    """Tác vụ 3: Giữ slot vận chuyển với đối tác giao vận."""
    print(f"[{time.strftime('%X')}] [Shipping] Đang kết nối API giao vận tới địa chỉ: {delivery_address}...")
    await asyncio.sleep(0.3)
    print(f"[{time.strftime('%X')}] [Shipping] LỖI: Kết nối socket bị từ chối!")
    raise ShippingServiceUnavailable(f"Máy chủ giao vận từ chối bắt tay TCP cho đơn {order_id}")

async def execute_checkout_pipeline(order_id: str, amount: float, address: str) -> None:
    print(f"\n=======================================================")
    print(f"BẮT ĐẦU XỬ LÝ CHECKOUT CHO ĐƠN HÀNG: {order_id}")
    print(f"=======================================================")
    start_timer = time.perf_counter()
    
    try:
        async with asyncio.TaskGroup() as tg:
            # Fork 3 tác vụ con đồng thời vào TaskGroup
            task_inv = tg.create_task(check_inventory(order_id))
            task_pay = tg.create_task(charge_credit_card(order_id, amount))
            task_ship = tg.create_task(reserve_shipping(order_id, address))
            
    except* PaymentGatewayTimeout as eg:
        # Bắt riêng nhánh lỗi thanh toán mà không làm gián đoạn việc bắt các nhánh lỗi khác
        print(f"\n[XỬ LÝ LỖI THANH TOÁN] Bắt được {len(eg.exceptions)} ngoại lệ PaymentGatewayTimeout:")
        for idx, exc in enumerate(eg.exceptions, 1):
            print(f"   [{idx}] Chi tiết lỗi: {exc}")
            
    except* ShippingServiceUnavailable as eg:
        # Bắt riêng nhánh lỗi dịch vụ giao vận
        print(f"\n[XỬ LÝ LỖI GIAO HÀNG] Bắt được {len(eg.exceptions)} ngoại lệ ShippingServiceUnavailable:")
        for idx, exc in enumerate(eg.exceptions, 1):
            print(f"   [{idx}] Chi tiết lỗi: {exc}")
            
    finally:
        total_duration = time.perf_counter() - start_timer
        print(f"\n[HOÀN TẤT] Quy trình checkout kết thúc an toàn sau {total_duration:.2f} giây.")

if __name__ == "__main__":
    asyncio.run(execute_checkout_pipeline(
        order_id="ORD-2026-XYZ",
        amount=2450000.0,
        address="Tòa nhà Bitexco, Quận 1, TP. Hồ Chí Minh"
    ))
```

### Phân tích kết quả thực thi
Khi các bạn chạy đoạn mã trên, một chuỗi sự kiện phối hợp tuyệt đẹp diễn ra:
1. Tại thời điểm $t = 0.3s$, `reserve_shipping` ném ra `ShippingServiceUnavailable`. Ngay lập tức `TaskGroup` gửi tín hiệu cancel tới `check_inventory` và `charge_credit_card`.
2. Tại thời điểm $t = 0.4s$, `charge_credit_card` cũng ném ra `PaymentGatewayTimeout`.
3. Tác vụ `check_inventory` (vốn dự định chạy 0.8s) lập tức nhận `CancelledError` tại $t \approx 0.3s-0.4s$, in thông báo giải phóng lock kho và kết thúc.
4. Cả hai khối `except* PaymentGatewayTimeout` và `except* ShippingServiceUnavailable` đều được kích hoạt độc lập để xử lý từng lỗi riêng biệt. Không có bất kỳ coroutine nào bị bỏ rơi chạy lén lút!

---

# IV. Lesson learned & Tổng kết: Best Practices & Cạm bẫy production

Qua quá trình triển khai thực tế trên các hệ thống microservices tải cao, mình rút ra một số kinh nghiệm "xương máu" mà các bạn cần lưu ý:

### 1. Cạm bẫy Await trực tiếp `tg.create_task()`
Một lỗi rất phổ biến khi lập trình viên chuyển từ `gather` sang `TaskGroup` là viết:
```python
# SAI LẦM PHỔ BIẾN:
async with asyncio.TaskGroup() as tg:
    res1 = await tg.create_task(service_a()) # Vô tình làm code chạy tuần tự!
    res2 = await tg.create_task(service_b())
```
Khi các bạn đặt `await` ngay trước `tg.create_task()`, Python sẽ dừng lại chờ `service_a` xong rồi mới khởi tạo `service_b`, phá vỡ hoàn toàn tính đồng quy song song.
- **Khắc phục**: Chỉ gọi `t1 = tg.create_task(...)`, sau đó để khối `async with` tự động join và thu hồi kết quả `t1.result()` sau khi thoát context.

### 2. Không bao giờ nuốt `asyncio.CancelledError`
Bên trong các hàm coroutine con, nếu các bạn dùng cấu trúc `try...except Exception:` để bắt lỗi, hãy yên tâm rằng `CancelledError` kế thừa từ `BaseException` nên sẽ không bị bắt. Tuy nhiên, nếu các bạn viết `try...except BaseException:` hoặc cố tình bắt `CancelledError` mà không `raise` lại, `TaskGroup` sẽ bị kẹt hoặc mất khả năng điều phối cancellation.

### 3. Tương thích Logging với hệ thống APM cũ
Các trình thu thập log cũ (ELK, Datadog formatter) chưa hỗ trợ bóc tách đệ quy của `ExceptionGroup`, dẫn tới việc log chỉ in ra một dòng cụt ngủn `ExceptionGroup: unhandled errors in a TaskGroup`.
- **Khắc phục**: Sử dụng `traceback.print_exception(eg)` trong Python 3.11+ để in toàn bộ cây phả hệ các exception con kèm đầy đủ traceback chi tiết của từng task.

### 4. Quy tắc vàng tổng kết
- Với mọi code viết cho **Python 3.11 trở lên**, hãy coi `asyncio.TaskGroup` là lựa chọn mặc định hàng đầu, thay thế hoàn toàn cho `asyncio.gather(return_exceptions=False)`.
- Cú pháp `except*` cho phép phân tách rành mạch trách nhiệm xử lý sự cố: lỗi mạng thì retry, lỗi validation thì báo người dùng, lỗi bảo mật thì alert PagerDuty — tất cả trong cùng một khối xử lý bất đồng bộ tao nhã.

Hy vọng bài viết này giúp các bạn tự tin làm chủ Structured Concurrency và nâng cấp chất lượng mã nguồn bất đồng bộ của mình lên một tầm cao mới!
