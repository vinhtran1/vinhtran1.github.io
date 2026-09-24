---
title: 'SQL Antipattern: Jaywalking - Sai lầm khi lưu danh sách phân tách bằng dấu phẩy'
date: 2026-09-24 09:00:00 +0700
categories: [Database, Best Practices]
tags: [SQL, Database Design, Antipattern, PostgreSQL, Jaywalking, 1NF, Junction Table, Foreign Key]
keywords: [Jaywalking, 1NF, Junction Table, Foreign Key]
pin: false
image:
  path: /assets/img/posts/2026/sql-antipattern-jaywalking/cover.webp
  alt: 'SQL Antipattern - Jaywalking: Phân tích và cách khắc phục'
---

# I. Dẫn nhập

Trong quá trình đi làm thực tế, chắc hẳn không ít lần các bạn đã từng bắt gặp một cấu trúc bảng cơ sở dữ liệu khiến bản thân cảm thấy "cấn cấn" ngay từ cái nhìn đầu tiên. Một trong những thiết kế kinh điển nhất mà mình từng gặp phải khi tối ưu hệ thống quản lý phân quyền người dùng có dạng như sau:

- Bảng `roles`: Định nghĩa các vai trò trong hệ thống.
- Bảng `users`: Chứa thông tin người dùng (`user_id`, `username`, `email`).
- Bảng `permissions`: Định nghĩa danh sách quyền hạn chi tiết (`permission_id`, `permission_name`).
- Cột `permission_ids` nằm ngay trong bảng `users`: Lưu trực tiếp một chuỗi các mã quyền cách nhau bởi dấu phẩy, ví dụ: `'1,2,5,8'`.

Khi lần đầu tiên nhìn thấy schema này, trực giác của một kỹ sư cơ sở dữ liệu sẽ gióng lên hồi chuông cảnh báo: Thiết kế này đã vi phạm trực tiếp **Dạng chuẩn 1 (First Normal Form - 1NF)** trong lý thuyết chuẩn hóa quan hệ. 

> **Dạng chuẩn 1 (1NF)** phát biểu ngắn gọn: Mọi thuộc tính trong một quan hệ phải có giá trị nguyên tố (Atomic Value) — tức là mỗi ô trong bảng chỉ được phép chứa duy nhất một giá trị đơn lẻ, không thể phân rã thêm, và tuyệt đối không được chứa danh sách, mảng hay tập hợp con.

Mãi cho đến khi mình tìm đọc cuốn sách kinh điển *"SQL Antipatterns: Avoiding the Pitfalls of Database Programming"* của tác giả **Bill Karwin**, mình mới biết thiết kế này có một tên gọi chính thức cực kỳ thú vị: **Jaywalking**.

Giải thích ngữ nghĩa một chút, trong tiếng Anh, *Jaywalking* là hành vi người đi bộ tùy tiện băng qua đường mà không tuân thủ vạch kẻ đường hoặc không đi qua các nút giao lộ an toàn. Tác giả Bill Karwin đã mượn hình tượng này để chỉ một thói quen rất phổ biến của lập trình viên: Thay vì tạo ra một bảng "giao lộ" trung gian (**Junction Table** hay Cross-reference Table) để kết nối hai thực thể có quan hệ Many-to-Many, lập trình viên lại "băng tắt" bằng cách nhồi nhét cả một danh sách giá trị vào cùng một cột dạng chuỗi.

Trong bài viết này, mình và các bạn sẽ cùng nhau mổ xẻ tại sao *Jaywalking* lại là một antipattern nguy hiểm, nó bóp nghẹt hiệu năng và tính toàn vẹn dữ liệu như thế nào, và cách giải quyết triệt để từ thiết kế schema chuẩn cho đến kỹ thuật viết SQL migration chuyển đổi dữ liệu an toàn.

---

# II. Kiến trúc / Nguyên lý

Nhìn qua thì việc lưu `'1,2,5,8'` vào một cột `VARCHAR` có vẻ rất tiện lợi: Bạn không cần tạo thêm bảng mới, không cần viết câu lệnh `JOIN` phức tạp khi lấy thông tin người dùng kèm quyền, và chỉ cần một câu `SELECT * FROM users` là lấy được hết. Thế nhưng, cái giá phải trả khi hệ thống lớn lên là vô cùng đắt đỏ. 

Dưới đây là 5 hệ lụy chí tử mà antipattern Jaywalking gây ra cho hệ thống cơ sở dữ liệu quan hệ:

### 1. Hiệu năng truy vấn chạm đáy (Vô hiệu hóa B-tree Index)

Giả sử bạn cần tìm tất cả người dùng đang nắm giữ quyền có `permission_id = 2`. Nếu cột lưu chuỗi `'1,2,5,8'`, bạn sẽ phải viết câu truy vấn tìm kiếm chuỗi con:

```sql
SELECT * FROM users WHERE permission_ids LIKE '%2%';
```

Câu lệnh trên dẫn đến hai vấn đề nghiêm trọng:
- **Kết quả sai lệch (False Positive):** Điều kiện `LIKE '%2%'` sẽ match trúng cả những user có quyền `12`, `25`, hay `102`. Để sửa chữa, lập trình viên thường phải "chắp vá" thêm dấu phẩy ở hai đầu:
  ```sql
  SELECT * FROM users WHERE ',' || permission_ids || ',' LIKE '%,2,%';
  ```
- **Buộc RDBMS phải quét toàn bộ bảng (Full Table Scan / Seq Scan):** Mẫu tìm kiếm có ký tự đại diện `%` ở đầu khiến cơ sở dữ liệu hoàn toàn không thể sử dụng cấu trúc B-tree Index. Cho dù bạn có đánh index trên cột `permission_ids`, bộ tối ưu (Optimizer) vẫn buộc phải duyệt qua từng dòng một và chạy hàm so khớp chuỗi trên CPU. Khi bảng `users` chạm mốc vài triệu bản ghi, độ trễ câu truy vấn sẽ tăng từ vài mili-giây lên hàng chục giây.

### 2. Đánh mất hoàn toàn tính toàn vẹn dữ liệu (Data Integrity)

Trong mô hình quan hệ, tính toàn vẹn được bảo vệ bởi **Ràng buộc khóa ngoại (Foreign Key Constraint)**. Tuy nhiên, RDBMS không thể thiết lập Foreign Key giữa một giá trị số và một chuỗi chứa nhiều số phân tách bằng dấu phẩy:
- Một user có thể lưu `permission_ids = '1,999,abc'` mà cơ sở dữ liệu không hề phản đối, dù quyền `999` chưa từng tồn tại.
- Khi một quyền bị xóa khỏi bảng `permissions` (`DELETE FROM permissions WHERE permission_id = 5`), hệ thống không thể tự động kích hoạt cơ chế `ON DELETE CASCADE` hoặc `RESTRICT`. Kết quả là dữ liệu "rác" (orphaned IDs) sẽ nằm vĩnh viễn trong cột chuỗi của bảng `users`.

### 3. Thao tác Cập nhật (INSERT / UPDATE / DELETE) biến thành cực hình

Hãy tưởng tượng bạn muốn thu hồi quyền số `2` của người dùng:
- Bạn không thể dùng một lệnh `DELETE` đơn giản mà phải dùng chuỗi các hàm biến đổi văn bản phức tạp:
  ```sql
  UPDATE users 
  SET permission_ids = TRIM(BOTH ',' FROM REPLACE(',' || permission_ids || ',', ',2,', ','))
  WHERE user_id = 42;
  ```
- Thao tác này tiêu tốn CPU, dễ sinh lỗi cú pháp khi chuỗi chỉ có một phần tử duy nhất, và đặc biệt dễ dẫn tới **Race Condition** khi hai tác vụ cùng đọc chuỗi cũ, biến đổi và ghi đè cùng một thời điểm.

### 4. Bế tắc khi tổng hợp (Aggregation) và Sắp xếp

- Muốn đếm xem mỗi người dùng có bao nhiêu quyền hạn? Với cấu trúc chuẩn, ta chỉ cần `COUNT(*)`. Với Jaywalking, bạn phải đếm số lượng dấu phẩy trong chuỗi (`LENGTH(col) - LENGTH(REPLACE(col, ',', '')) + 1`), một thủ thuật chắp vá tốn kém.
- Muốn thống kê xem quyền nào đang được gán cho nhiều người dùng nhất (`GROUP BY permission_id`)? Bạn hoàn toàn bó tay nếu không dùng các hàm phân tách chuỗi phức tạp.

### 5. Giới hạn độ dài cột (Length Limits)

Nếu bạn khai báo cột là `VARCHAR(255)`, khi danh sách quyền của một người dùng vượt quá độ dài này, câu lệnh cập nhật sẽ ném ra ngoại lệ `String data, right truncated`. Việc mở rộng lên `TEXT` chỉ là dời cái bẫy sang một vùng rủi ro khác lớn hơn về quản lý bộ nhớ đệm (buffer cache) và bộ nhớ I/O (TOAST storage).

---

# III. Cài đặt / Hands-on code

Để giải quyết dứt điểm vấn đề này, chúng ta sẽ cùng đi qua 4 bước: Thiết lập schema chuẩn, đối chiếu kế hoạch thực thi (EXPLAIN plan), thực hiện migration dữ liệu sống và thảo luận các trường hợp ngoại lệ.

### Bước 1: Chuẩn hóa Schema bằng Junction Table

Thay vì gộp danh sách ID vào bảng `users`, ta tách mối quan hệ Many-to-Many sang một bảng giao lộ trung gian là `user_permissions`:

```sql
-- 1. Bảng thực thể Users
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp()
);

-- 2. Bảng thực thể Permissions
CREATE TABLE permissions (
    permission_id SERIAL PRIMARY KEY,
    permission_name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT
);

-- 3. Junction Table chuẩn hóa (Bảng giao lộ)
CREATE TABLE user_permissions (
    user_id INT NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    permission_id INT NOT NULL REFERENCES permissions(permission_id) ON DELETE CASCADE,
    assigned_at TIMESTAMPTZ DEFAULT clock_timestamp(),
    PRIMARY KEY (user_id, permission_id)
);

-- Tạo Index hỗ trợ truy vấn ngược: tìm tất cả user có cùng một permission
CREATE INDEX idx_user_permissions_perm_id ON user_permissions (permission_id);
```

Với cấu trúc này:
- Mỗi ô chỉ chứa một giá trị nguyên tử (đạt chuẩn 1NF).
- Khóa chính phức hợp `PRIMARY KEY (user_id, permission_id)` đảm bảo một user không bao giờ bị gán trùng lặp một quyền hai lần.
- Ràng buộc `REFERENCES ... ON DELETE CASCADE` đảm bảo tính toàn vẹn tham chiếu 100%.

### Bước 2: So sánh hiệu năng thực tế qua EXPLAIN ANALYZE

Hãy cùng so sánh hai câu truy vấn khi tìm kiếm tất cả user có quyền `permission_id = 2` trên cơ sở dữ liệu có 100,000 dòng.

**Trường hợp Antipattern (Jaywalking):**
```sql
EXPLAIN ANALYZE
SELECT user_id, username 
FROM legacy_users 
WHERE ',' || permission_ids || ',' LIKE '%,2,%';
```

Kế hoạch thực thi trả về:
```
Seq Scan on legacy_users  (cost=0.00..2950.00 rows=500 width=40) (actual time=0.045..28.450 rows=480 loops=1)
  Filter: ((',' || permission_ids || ',') ~~ '%,2,%'::text)
  Rows Removed by Filter: 99520
Planning Time: 0.120 ms
Execution Time: 28.510 ms
```

**Trường hợp chuẩn hóa với Junction Table:**
```sql
EXPLAIN ANALYZE
SELECT u.user_id, u.username
FROM users u
JOIN user_permissions up ON u.user_id = up.user_id
WHERE up.permission_id = 2;
```

Kế hoạch thực thi trả về:
```
Nested Loop  (cost=0.57..82.40 rows=480 width=40) (actual time=0.021..0.450 rows=480 loops=1)
  ->  Bitmap Heap Scan on user_permissions up  (cost=4.28..25.10 rows=480 width=4)
        Recheck Cond: (permission_id = 2)
        ->  Bitmap Index Scan on idx_user_permissions_perm_id  (cost=0.00..4.16 rows=480 width=0)
  ->  Index Scan using users_pkey on users u  (cost=0.29..0.35 rows=1 width=40)
        Index Cond: (user_id = up.user_id)
Planning Time: 0.150 ms
Execution Time: 0.520 ms
```

Tốc độ cải thiện hơn **50 lần** (từ 28.5ms xuống 0.52ms). Ở quy mô hàng triệu bản ghi, sự chênh lệch này là ranh giới giữa một trang web phản hồi tức thì và một hệ thống bị timeout connection pool.

### Bước 3: Migration dữ liệu từ Jaywalking sang Junction Table

Nếu dự án của các bạn đang dính phải Jaywalking, làm thế nào để di chuyển dữ liệu sang bảng mới mà không cần viết script Python hay NodeJS bên ngoài?

Trong PostgreSQL, chúng ta có thể sử dụng kết hợp hai hàm cực mạnh: `string_to_array` và `unnest`:

```sql
-- Chuyển đổi dữ liệu tự động 100% bằng SQL
INSERT INTO user_permissions (user_id, permission_id)
SELECT 
    legacy.user_id,
    p_id::INT AS permission_id
FROM legacy_users legacy
CROSS JOIN LATERAL unnest(string_to_array(legacy.permission_ids, ',')) AS p_id
WHERE legacy.permission_ids IS NOT NULL 
  AND TRIM(legacy.permission_ids) != ''
  AND p_id ~ '^[0-9]+$' -- Đảm bảo chỉ parse các giá trị số hợp lệ
ON CONFLICT (user_id, permission_id) DO NOTHING;
```

Sau khi di chuyển dữ liệu thành công và đối soát số lượng bản ghi đầy đủ, các bạn có thể an tâm drop cột `permission_ids` cũ:
```sql
ALTER TABLE legacy_users DROP COLUMN permission_ids;
```

### Bước 4: Khi nào Jaywalking được chấp nhận (Edge Cases & Giải pháp thay thế)?

Không có thiết kế nào là sai tuyệt đối 100% trong mọi ngữ cảnh kỹ thuật. Tác giả Bill Karwin cũng chỉ ra một số trường hợp ngoại lệ:
1. **Thuộc tính hiển thị thuần túy (Read-only UI attributes):** Dữ liệu chỉ dùng để hiển thị lên màn hình, không bao giờ tham gia vào mệnh đề `WHERE`, không cần `JOIN`, không bao giờ aggregate hay search.
2. **PostgreSQL Hiện đại: Kiểu dữ liệu mảng hoặc JSONB có đánh GIN Index:**
   Nếu các bạn thực sự muốn lưu danh sách phần tử trong một cột vì lý do denormalization cho hiệu năng đọc cực độ, hãy dùng kiểu dữ liệu chuẩn của PostgreSQL:
   ```sql
   ALTER TABLE users ADD COLUMN permission_ids INT[];
   CREATE INDEX idx_users_perms_gin ON users USING GIN (permission_ids);

   -- Tìm kiếm bằng toán tử mảng chứa (@>) tận dụng GIN Index cực nhanh:
   SELECT * FROM users WHERE permission_ids @> ARRAY[2];
   ```
   Giải pháp này vừa giữ được tính nguyên tố của phần tử trong mảng, vừa tận dụng được GIN Index để tăng tốc độ truy vấn, thay vì chuỗi thuần túy như Jaywalking.

---

# IV. Lesson learned / Tổng kết

Thiết kế cơ sở dữ liệu quan hệ giống như việc xây dựng móng nhà: Một vết nứt nhỏ trong khâu chuẩn hóa có thể dẫn đến sự sụp đổ hiệu năng của toàn bộ hệ thống sau vài năm vận hành.

Tổng kết lại những bài học kinh nghiệm (Lesson Learned) mà mình muốn chia sẻ với các bạn:

1. **Tuân thủ chuẩn 1NF:** Luôn đảm bảo mỗi cột chỉ chứa một giá trị nguyên tử đơn lẻ. Đừng bao giờ lưu chuỗi phân cách bằng dấu phẩy để đại diện cho một danh sách.
2. **Junction Table là giải pháp chân ái:** Cho quan hệ nhiều-nhiều (Many-to-Many), việc tạo bảng trung gian với khóa ngoại kép là giải pháp duy nhất đảm bảo tính toàn vẹn tham chiếu và cho phép RDBMS tận dụng tối đa Index.
3. **Tuyệt đối tránh so khớp chuỗi trên ID:** Biến đổi khóa quan hệ thành chuỗi để `LIKE` hay `REGEX` sẽ bóp nghẹt B-tree Index và biến mọi truy vấn thành Full Table Scan.
4. **Tận dụng tính năng Native của RDBMS khi cần Denormalization:** Nếu bắt buộc phải gộp dữ liệu vì bài toán đặc thù, hãy dùng kiểu dữ liệu `ARRAY` hoặc `JSONB` kết hợp với `GIN Index` trong PostgreSQL thay vì chuỗi text thô sơ.

Hy vọng bài viết này giúp các bạn nhận diện sớm và tự tin loại bỏ antipattern Jaywalking trong các dự án thực tế. Ở bài viết tiếp theo, mình sẽ cùng các bạn khám phá sâu hơn về cách tối ưu các truy vấn ngắn thông qua Index Selectivity và Execution Plan trong PostgreSQL!
