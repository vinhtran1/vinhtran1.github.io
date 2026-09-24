---
title: 'Giải quyết xung đột ghi trong Apache Cassandra: Last-Write-Wins (LWW), hiểm họa NTP Clock Skew và Paxos LWT'
date: 2026-09-24 12:00:00 +0700
categories: [Distributed Systems, Database Internals]
tags: [Apache Cassandra, NoSQL, Distributed Systems, Database Internals, Consensus Algorithms]
keywords: [Apache Cassandra, NoSQL, Distributed Systems, Database Internals]
pin: false
image:
  path: /assets/img/posts/2026/giai-quyet-xung-dot-ghi-trong-apache-cassandra-last-write-wins-va-dong-ho-ntp/cover.webp
  alt: 'Cơ chế giải quyết xung đột Last-Write-Wins trong Apache Cassandra và hiểm họa NTP Clock Skew'
---

# I. Dẫn nhập: Đặt vấn đề và Thách thức kỹ thuật

Chào các bạn, trong thế giới cơ sở dữ liệu phân tán (Distributed Databases), **Apache Cassandra** luôn được ca tụng là cỗ máy ghi dữ liệu vô song. Với kiến trúc Masterless (không có Master hay Slave, mọi node bình đẳng như nhau) kế thừa từ bài báo khoa học Dynamo kinh điển của Amazon và mô hình lưu trữ Bigtable của Google, Cassandra có thể dễ dàng nuốt trọn hàng triệu phép ghi mỗi giây trên một cụm gồm hàng trăm máy chủ vật lý đặt tại nhiều trung tâm dữ liệu (Multi-Datacenter) trải rộng toàn cầu.

Bất kỳ node nào trong cụm cũng có thể đóng vai trò **Coordinator** để tiếp nhận lệnh đọc/ghi từ client. Không có khóa bi quan (Pessimistic Locking), không có giao thức Two-Phase Commit (2PC) làm chậm đường truyền mạng. Hệ thống tối ưu hóa hoàn toàn cho Tính sẵn sàng (High Availability) và Khả năng chịu lỗi phân vùng mạng (Partition Tolerance) theo định lý CAP.

Thế nhưng, tự do nào cũng đi kèm với một cái giá phải trả. Hãy tưởng tượng một tình huống thực tế kinh điển trong hệ thống phân tán:
- Một tài khoản ngân hàng của khách hàng có số dư hiện tại là `$1,000`.
- Khách hàng thực hiện nạp `$200` tại một máy ATM ở Tokyo $\rightarrow$ Lệnh ghi gửi đến Datacenter Châu Á tại thời điểm $t_1$.
- Cùng lúc đó, hệ thống thanh toán tự động tại New York trừ `$100` tiền phí dịch vụ $\rightarrow$ Lệnh ghi gửi đến Datacenter Bắc Mỹ tại thời điểm $t_2$.
- Do độ trễ mạng Internet giữa hai bờ đại dương mất khoảng $150\text{ms}$, hai bản ghi này đến các node bản sao (Replicas) theo thứ tự đảo ngược nhau!

```
+-----------------------------------------------------------------------------+
|                     NGHỊCH LÝ XUNG ĐỘT GHI TRONG CASSANDRA                  |
|                                                                             |
|  Tokyo Node (Asia DC)                               New York Node (US DC)   |
|  [Update Balance = $1,200]                          [Update Balance = $900] |
|             \                                             /                 |
|              \                                           /                  |
|               v                                         v                   |
|           +-------------------------------------------------+               |
|           |             CASSANDRA REPLICA NODE              |               |
|           |                                                 |               |
|           |  - Ai đến trước? Ai đến sau?                    |               |
|           |  - Dựa vào cái gì để phân xử thắng thua?        |               |
|           |                                                 |               |
|           |  ---> Cơ chế: LAST-WRITE-WINS (LWW)             |               |
|           |       Dựa 100% vào Physical Timestamp!          |               |
|           +-------------------------------------------------+               |
|                                     |                                       |
|             HIỂM HỌA: Đồng hồ vật lý bị lệch (NTP Clock Skew)               |
|             -> Mất mát dữ liệu âm thầm (Silent Data Overwrite)!             |
+-----------------------------------------------------------------------------+
```

Để giải quyết xung đột khi hai hay nhiều thao tác ghi cùng nhắm vào một bản ghi mà không cần phải khóa bảng, Apache Cassandra áp dụng một cơ chế mặc định mang tên: **Last-Write-Wins (LWW)** ở cấp độ từng Cell (Column).

Và đây chính là nơi cơn ác mộng kỹ thuật bắt đầu!
Cơ chế LWW phụ thuộc hoàn toàn vào **Thời gian vật lý (Physical Wall-clock Time)** được gắn nhãn trên mỗi bản ghi. Trong thế giới thực, các tinh thể thạch anh dao động trong bo mạch máy chủ luôn bị trôi (Drift) do nhiệt độ và tuổi thọ linh kiện. Nếu giao thức đồng bộ đồng hồ mạng (NTP) bị gián đoạn hoặc bị lệch dù chỉ vài chục mili-giây, dữ liệu mới hơn có thể bị xóa sổ vĩnh viễn bởi dữ liệu cũ hơn, hoặc một bản ghi đã bị xóa (`Tombstone`) có thể bất ngờ "sống lại" như một bóng ma (Zombie Data)!

Trong bài viết chuyên sâu này, mình sẽ cùng các bạn bóc tách tận gốc cơ chế Cell-level LWW của Cassandra, viết code mô phỏng chính xác thảm họa mất dữ liệu do Clock Skew gây ra, và xây dựng 3 lớp phòng thủ kiên cố: từ việc chuẩn hóa đồng bộ đồng hồ bằng `chrony`, ứng dụng giao thức đồng thuận **Paxos** trong **Lightweight Transactions (LWT)**, cho đến tư duy thiết kế dữ liệu bất biến (Immutability).

---

# II. Kiến trúc / Nguyên lý cốt lõi

### 1. Mô hình lưu trữ Cell-Level và Bản chất của Last-Write-Wins (LWW)

Trong cơ sở dữ liệu quan hệ truyền thống (như PostgreSQL hay MySQL), một phép `UPDATE` thường tác động lên toàn bộ dòng (Row). Nhưng trong Apache Cassandra (cấu trúc lưu trữ LSM-Tree), mỗi dòng thực chất là một tập hợp các **Cell** (tương ứng với từng cột dữ liệu) hoàn toàn độc lập:

Mỗi Cell lưu trữ cấu trúc nhị phân gồm 4 trường thông tin:
$$\text{Cell} = \{ \text{Column Name}, \text{Value}, \text{Timestamp (Microseconds)}, \text{TTL / Deletion Time} \}$$

```
 BẢN GHI DỮ LIỆU CỦA MỘT DÒNG TRONG CASSANDRA:
 Row Key: user_id = 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d
 +---------------------------------------------------------------------------+
 | Cell 1: Column "email"                                                    |
 | Value: "vinh@domain.com"                                                  |
 | Timestamp: 1727172000000000 (Microseconds)                                |
 +---------------------------------------------------------------------------+
 | Cell 2: Column "status"                                                   |
 | Value: "ACTIVE"                                                           |
 | Timestamp: 1727172005000000 (Microseconds)                                |
 +---------------------------------------------------------------------------+
 | Cell 3: Column "balance"                                                  |
 | Value: 1200.50                                                            |
 | Timestamp: 1727172010000000 (Microseconds)                                |
 +---------------------------------------------------------------------------+
```

Khi có hai phép ghi cạnh tranh cùng cập nhật vào một Cell trên các node bản sao khác nhau, Cassandra giải quyết xung đột theo thuật toán xác định (Deterministic Algorithm):

1. **So sánh Timestamp**: Cell nào có `Timestamp` (tính bằng microsecond) lớn hơn sẽ **THẮNG** (`Winner = max(timestamp_A, timestamp_B)`). Cell có timestamp nhỏ hơn sẽ bị loại bỏ hoàn toàn trong quá trình **Read Repair** hoặc **SSTable Compaction**.
2. **Tie-breaker (Khi Timestamp bằng nhau tuyệt đối)**: Nếu hai phép ghi ngẫu nhiên có cùng một timestamp microsecond, Cassandra sẽ so sánh mảng byte nhị phân của giá trị dữ liệu (`byte-wise comparison of Values`). Giá trị nào lớn hơn về mặt từ điển nhị phân sẽ thắng!
3. **Cập nhật độc lập không xung đột (Fine-grained Concurrency)**: Nếu Transaction A cập nhật cột `email`, trong khi Transaction B cập nhật cột `phone_number` tại cùng một thời điểm, cả hai Cell đều được lưu trữ hoàn hảo mà không hề đè lên nhau, vì chúng là hai Cell riêng biệt trong LSM-tree.

### 2. Hiểm họa NTP Clock Skew: Khi đồng hồ vật lý đánh lừa hệ thống

Vấn đề cốt tử nằm ở chỗ: **Timestamp lấy từ đâu?**
Theo mặc định, timestamp được tạo bởi **Client Driver** tại thời điểm gửi request, hoặc do **Coordinator Node** gán vào khi nhận được request nếu client không chỉ định. Cả hai nguồn này đều dựa vào đồng hồ hệ thống vật lý (Wall-clock Time) của máy chủ!

Hãy phân tích hai kịch bản thảm họa kinh hoàng nhất trong production:

#### Kịch bản 1: Mất dữ liệu âm thầm (Silent Data Overwrite)
- Máy chủ **Node A** (Coordinator 1) bị lỗi drift đồng hồ, chạy **nhanh hơn 100ms** so với giờ chuẩn quốc tế ($t_{skew} = t_{real} + 100\text{ms}$).
- Máy chủ **Node B** (Coordinator 2) chạy **chuẩn giờ**.
- Vào lúc $12:00:00.000$, Node A xử lý yêu cầu đổi trạng thái đơn hàng thành `status = 'SHIPPED'`. Nó gắn nhãn timestamp: `12:00:00.100`.
- Vào lúc $12:00:00.050$ (50ms sau), khách hàng phát hiện nhầm lẫn và bấm hủy đơn hàng. Request đến Node B cập nhật `status = 'CANCELLED'`. Node B gắn nhãn timestamp đúng: `12:00:00.050`.
- Khi bản ghi từ Node B được đồng bộ sang các bản sao, Cassandra đem ra so sánh:
  $$12:00:00.100 (\text{SHIPPED}) > 12:00:00.050 (\text{CANCELLED})$$
- **Kết quả đau đớn**: Dù hành động hủy đơn diễn ra SAU, nhưng giá trị `SHIPPED` cũ kỹ lại THẮNG! Hệ thống âm thầm giữ nguyên trạng thái giao hàng, xóa sạch yêu cầu hủy đơn của khách hàng mà không hề phát ra bất kỳ thông báo lỗi nào!

#### Kịch bản 2: Bóng ma dữ liệu sống lại (Zombie Data via Tombstones)
Trong Cassandra, lệnh `DELETE` không thực sự xóa dữ liệu trên đĩa ngay lập tức mà ghi đè một Cell đặc biệt gọi là **Tombstone** mang timestamp của thời điểm xóa.
Nếu một thao tác ghi dữ liệu trước đó vô tình mang timestamp trong tương lai (do máy client chạy sai giờ trước 1 ngày), thì sau đó dù bạn có gọi `DELETE` bao nhiêu lần, Tombstone (mang timestamp hiện tại) sẽ luôn **NHỎ HƠN** timestamp tương lai của dữ liệu. Kết quả là dữ liệu bị xóa sẽ tiếp tục "đội mồ sống dậy" sau mỗi đợt compaction!

```
 THẢM HỌA CLOCK SKEW VÀ TOMBSTONE:
 
 1. INSERT mang timestamp tương lai (do lỗi NTP):
    [user_id: 1, name: 'Alice']  -> Timestamp: 2026-09-25 00:00:00
 
 2. DELETE chạy vào ngày hôm nay:
    [user_id: 1, TOMBSTONE]      -> Timestamp: 2026-09-24 12:00:00
 
 3. Cassandra đối soát LWW:
    2026-09-25 > 2026-09-24  ===> DỮ LIỆU THẮNG TOMBSTONE!
    -> user_id 1 KHÔNG THỂ BỊ XÓA! (Zombie Data)
```

### 3. Giải pháp cấp cao: Lightweight Transactions (LWT) dựa trên Paxos

Khi nghiệp vụ bắt buộc phải đảm bảo tính nhất quán tuần tự tuyệt đối (Linearizable Consistency) — ví dụ: tạo tài khoản với username duy nhất (`INSERT ... IF NOT EXISTS`), hoặc kiểm tra số dư trước khi trừ (`UPDATE ... IF balance >= 100`) — Cassandra cung cấp giải pháp **Lightweight Transactions (LWT)** dựa trên thuật toán đồng thuận **Paxos**.

Giao thức Paxos trong Cassandra gồm 4 pha mạng liên tiếp:

```
 Coordinator Node                                    Quorum Replicas
        |                                                   |
        | ----- 1. PREPARE (Ballot Number) ---------------> |
        | <---- 1. PROMISE (Highest Ballot Seen) ---------- |
        |                                                   |
        | ----- 2. READ / PROPOSE (New Value Proposed) ---> |
        | <---- 2. ACCEPT (Replica Agrees) ---------------- |
        |                                                   |
        | ----- 3. COMMIT (Finalize Value) ---------------> |
        | <---- 3. ACK (Acknowledge Commit) --------------- |
        |                                                   |
        | ----- 4. LEARN (Notify all learners) -----------> |
        v                                                   v
```

1. **Prepare / Promise**: Coordinator sinh một Ballot ID đơn điệu tăng dần (sử dụng UUIDv1 kết hợp clock sequencing). Các Replicas cam kết từ chối bất kỳ Ballot nào cũ hơn.
2. **Read / Propose**: Coordinator đọc trạng thái dữ liệu hiện tại từ đa số Quorum Replicas, kiểm tra điều kiện `IF`. Nếu thỏa mãn, đề xuất giá trị mới.
3. **Accept / Commit**: Các Replicas ghi nhận giá trị đề xuất vào bảng trạng thái Paxos tạm thời.
4. **Learn**: Ghi chính thức vào bảng dữ liệu chính.

**Đánh đổi (Trade-off)**:
LWT không phụ thuộc vào đồng hồ NTP để phân xử xung đột, nhưng nó tốn tới **4 round-trips mạng**. Thông lượng ghi của LWT giảm khoảng **75%** so với lệnh ghi thông thường sử dụng LWW.

---

# III. Cài đặt / Hands-on code & Tối ưu thực chiến

### 1. Kịch bản Python tái hiện Thảm họa Clock Skew bằng `USING TIMESTAMP`

Cassandra Query Language (CQL) cho phép chúng ta can thiệp trực tiếp timestamp của phép ghi thông qua mệnh đề `USING TIMESTAMP <microseconds>`. Chúng ta sẽ tận dụng tính năng này để mô phỏng chính xác sự cố trôi đồng hồ mạng trong thực tế:

```python
import time
import uuid
from cassandra.cluster import Cluster
from cassandra.query import SimpleStatement
from cassandra import ConsistencyLevel

def demonstrate_clock_skew():
    # Kết nối tới cụm Cassandra
    cluster = Cluster(["127.0.0.1"], port=9042)
    session = cluster.connect()

    # 1. Khởi tạo Keyspace và Bảng kiểm thử
    session.execute("""
        CREATE KEYSPACE IF NOT EXISTS bank_demo 
        WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};
    """)
    session.execute("""
        CREATE TABLE IF NOT EXISTS bank_demo.account_balance (
            account_id uuid PRIMARY KEY,
            balance decimal,
            last_op text
        );
    """)

    acc_id = uuid.uuid4()
    base_micro = int(time.time() * 1_000_000)

    # 2. Thao tác 1 (Sự kiện xảy ra trước, nhưng node bị Clock Drift chạy trước 5 GIÂY)
    skewed_future_ts = base_micro + 5_000_000  # +5 giây trong tương lai
    print(f"[*] Node A (Clock Skewed +5s) ghi: balance = 500.0, ts = {skewed_future_ts}")
    stmt1 = SimpleStatement(
        f"INSERT INTO bank_demo.account_balance (account_id, balance, last_op) "
        f"VALUES ({acc_id}, 500.0, 'DEPOSIT_FROM_NODE_A') "
        f"USING TIMESTAMP {skewed_future_ts};",
        consistency_level=ConsistencyLevel.ONE
    )
    session.execute(stmt1)

    # Giả lập trễ 1 giây trong thế giới thực
    time.sleep(1.0)

    # 3. Thao tác 2 (Sự kiện diễn ra SAU trong thực tế, nhưng node chạy chuẩn giờ)
    real_time_now_ts = base_micro + 1_000_000  # +1 giây thực tế
    print(f"[*] Node B (Đúng giờ thực tế) ghi: balance = 200.0, ts = {real_time_now_ts}")
    stmt2 = SimpleStatement(
        f"UPDATE bank_demo.account_balance "
        f"USING TIMESTAMP {real_time_now_ts} "
        f"SET balance = 200.0, last_op = 'WITHDRAW_FROM_NODE_B' "
        f"WHERE account_id = {acc_id};",
        consistency_level=ConsistencyLevel.ONE
    )
    session.execute(stmt2)

    # 4. Kiểm tra kết quả trong Cassandra
    row = session.execute(f"""
        SELECT balance, last_op, WRITETIME(balance), WRITETIME(last_op) 
        FROM bank_demo.account_balance 
        WHERE account_id = {acc_id};
    """).one()

    print("\n" + "=" * 60)
    print(" KẾT QUẢ ĐỐI SOÁT TRONG CƠ SỞ DỮ LIỆU:")
    print(f" - Số dư thực tế hiển thị: {row.balance}")
    print(f" - Thao tác thắng:        {row.last_op}")
    print(f" - Writetime của Cell:     {row.writetime_balance}")
    print("=" * 60)
    
    # Assert chứng minh sự sai lệch
    if str(row.last_op) == 'DEPOSIT_FROM_NODE_A':
        print("[!] BÁO ĐỘNG ĐỎ: Thao tác cũ hơn đã ghi đè thao tác mới do Clock Skew!")

    cluster.shutdown()

if __name__ == "__main__":
    demonstrate_clock_skew()
```

### 2. Thực thi Lightweight Transactions (LWT) chống Race Condition

Để ngăn chặn hoàn toàn việc mất mát dữ liệu do xung đột đồng thời, chúng ta sử dụng mệnh đề so sánh điều kiện nguyên tử `IF`:

```python
def safe_balance_update(session, acc_id, expected_balance, new_balance):
    """
    Thực thi giao dịch LWT Paxos: Chỉ cập nhật nếu số dư hiện tại bằng đúng expected_balance.
    """
    query = """
        UPDATE bank_demo.account_balance 
        SET balance = %s, last_op = 'SAFE_LWT_UPDATE'
        WHERE account_id = %s 
        IF balance = %s;
    """
    prepared = session.prepare(query)
    prepared.consistency_level = ConsistencyLevel.QUORUM

    # Kết quả trả về chứa cột đặc biệt: [applied] (boolean)
    result = session.execute(prepared, (new_balance, acc_id, expected_balance))
    row = result.one()
    
    if row.applied:
        print(f"[OK] Cập nhật số dư thành công lên {new_balance} qua Paxos LWT.")
        return True
    else:
        print(f"[FAIL] Giao dịch thất bại! Số dư hiện tại trên DB là {row.balance} thay vì {expected_balance}.")
        return False
```

### 3. Cấu hình Chuẩn hóa Đồng bộ Thời gian với `chrony` trên AWS/Linux

Để triệt tiêu hiện tượng Clock Drift trong hạ tầng cơ sở dữ liệu Cassandra, toàn bộ máy chủ phải được cấu hình đồng bộ thời gian thông qua dịch vụ **AWS Time Sync Service** (hoặc PTP Stratum-1 server) với phần mềm `chrony`:

File cấu hình `/etc/chrony/chrony.conf`:

```conf
# Sử dụng AWS Time Sync Service IP nội bộ (Stratum-1 PTP source)
server 169.254.169.123 prefer iburst minpoll 4 maxpoll 4

# Ghi nhật ký độ lệch thời gian
driftfile /var/lib/chrony/drift

# Cho phép bước nhảy đồng hồ (step) chỉ trong 3 lần cập nhật đầu tiên nếu lệch > 0.1s
# Sau đó TUYỆT ĐỐI KHÔNG STEP (tránh nhảy giật lùi thời gian phá hỏng LWW)
makestep 0.1 3

# Kích hoạt chế độ Slew: điều chỉnh tần số đồng hồ mượt mà thay vì nhảy cóc
maxupdateskew 100.0

# Ghi nhận log phục vụ giám sát Prometheus/Datadog
logdir /var/log/chrony
log measurements statistics tracking
```

Lệnh kiểm tra độ lệch thời gian (Offset) trên từng node:
```bash
# Kiểm tra độ lệch chuẩn xác
chronyc tracking | grep -E "Reference ID|System time|RMS offset"
```
Kết quả mong muốn: `RMS offset` phải luôn nhỏ hơn **0.000500 giây (0.5 ms)**.

### 4. Thiết kế Mô hình Dữ liệu Bất biến (Append-Only Event Sourcing)

Phương thuốc chữa bách bệnh cho bài toán xung đột ghi trong hệ thống phân tán không phải là cố gắng tìm cách "sửa" một Cell có sẵn, mà là chuyển dịch tư duy sang **Dữ liệu bất biến (Immutability)**:

```sql
CREATE KEYSPACE IF NOT EXISTS ledger_system 
WITH replication = {'class': 'NetworkTopologyStrategy', 'us-east': 3, 'ap-northeast': 3};

-- Bảng sổ cái bất biến: Chỉ INSERT, tuyệt đối không UPDATE
CREATE TABLE ledger_system.account_transactions (
    account_id uuid,
    transaction_id timeuuid,  -- UUIDv1 chứa timestamp đơn điệu và MAC address
    amount decimal,
    transaction_type text,    -- 'CREDIT' hoặc 'DEBIT'
    payload_metadata text,
    PRIMARY KEY ((account_id), transaction_id)
) WITH CLUSTERING ORDER BY (transaction_id DESC);
```

Bằng cách dùng `timeuuid` làm clustering key, mỗi giao dịch là một bản ghi mới toanh được chèn vào. Bất kỳ node nào ghi trước hay sau đều nằm ở một vị trí xác định trong cây sắp xếp. Xung đột ghi giảm về con số **0**!

---

# IV. Lesson learned / Tổng kết & Best Practices

Qua quá trình vận hành các cụm Apache Cassandra quy mô lớn trong môi trường tài chính và viễn thông, mình xin chia sẻ 5 nguyên tắc sống còn khi giải quyết xung đột ghi:

### 1. Tuyệt đối không để Client Application tự sinh Timestamp
Mặc dù các Cassandra Driver thường mặc định dùng đồng hồ client (`client-side timestamp generator`), nhưng máy người dùng hoặc các container microservices thường có độ lệch đồng hồ rất lớn. Luôn cấu hình Driver sử dụng Server-side timestamping hoặc đảm bảo toàn bộ worker nodes chạy trong một Kubernetes cluster được đồng bộ `chrony` chuẩn xác dưới 1ms.

### 2. Giám sát độ lệch thời gian (NTP Clock Offset) bằng cảnh báo P1
Hãy đưa metric `node_timex_offset_seconds` của Prometheus Node Exporter lên bảng điều khiển trung tâm. Nếu bất kỳ node nào trong cluster có độ lệch đồng hồ vượt quá **20 mili-giây**, hệ thống phải ngay lập tức kích hoạt cảnh báo mức độ cao (P1 Alert) và tạm thời rút node đó ra khỏi vòng quay xử lý để tránh làm nhiễm độc timestamp vào SSTables.

### 3. Sử dụng Paxos LWT một cách tiết kiệm và có chủ đích
Paxos rất an toàn nhưng tốn kém tài nguyên mạng gấp 4 lần. Hãy chỉ sử dụng LWT cho các tác vụ quan trọng yêu cầu trạng thái duy nhất (Unique Constraints) hoặc máy trạng thái hữu hạn (State Machine Transition). Đừng bao giờ lạm dụng `IF` cho các luồng ghi telemetry hay log thông thường.

### 4. Hiểu rõ giới hạn của Cassandra Counter
Nếu các bạn cần đếm lượt view, like hay số dư lũy kế, hãy cân nhắc sử dụng kiểu dữ liệu `counter` của Cassandra. Kiểu dữ liệu này được xây dựng trên nền tảng CRDT (Conflict-free Replicated Data Types - PN-Counter), cho phép các phép cộng/trừ giao hoán tự do mà không phụ thuộc vào thứ tự thời gian LWW.

### 5. Tư duy "Immutability Wins"
Bất cứ khi nào có thể, hãy thiết kế bảng dữ liệu theo dạng **Append-Only Event Store**. Thay vì liên tục đè lên một dòng dữ liệu, hãy ghi lại lịch sử các sự kiện biến đổi. Dữ liệu bất biến là vũ khí tối thượng giúp loại bỏ 100% rủi ro xung đột ghi, đem lại khả năng kiểm toán hoàn hảo và giúp hệ thống phân tán của các bạn vững vàng trước mọi cơn bão đồng hồ!
