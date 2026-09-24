---
title: 'Cải thiện tìm kiếm trên PostgreSQL: Tận dụng Full Text Search thay thế LIKE và Elasticsearch'
date: 2026-09-24 09:30:00 +0700
categories: [Hands-On, Database]
tags: [PostgreSQL, Full Text Search, tsvector, tsquery, GIN Index, plainto_tsquery, Database]
keywords: [tsvector, tsquery, GIN Index, plainto_tsquery, Full Text Search]
pin: false
image:
  path: /assets/img/posts/2026/postgresql-full-text-search/cover.webp
  alt: 'PostgreSQL Full Text Search: Biến PostgreSQL thành search engine hiệu năng cao'
---

# I. Dẫn nhập

Khi phát triển bất kỳ ứng dụng nào — từ trang thương mại điện tử, ứng dụng đọc tin tức cho đến blog cá nhân — tính năng tìm kiếm văn bản luôn là một trong những chức năng được người dùng sử dụng nhiều nhất. Khi người dùng nhập một vài từ khóa vào thanh tìm kiếm, họ kỳ vọng kết quả trả về phải nhanh chóng (trong vòng vài chục mili-giây) và các bài viết hoặc sản phẩm liên quan nhất phải xuất hiện ngay ở đầu danh sách.

Trong thực tế đi làm, mình nhận thấy các team phát triển phần mềm thường đứng trước 3 sự lựa chọn quen thuộc:

1. **Phương pháp "Mì ăn liền": Toán tử `LIKE` hoặc `ILIKE`**
   - Rất dễ viết: `WHERE title ILIKE '%keyword%' OR content ILIKE '%keyword%'`.
   - **Nhược điểm chí mạng:** Với toán tử có dấu `%` ở đầu, cơ sở dữ liệu bị tê liệt B-tree Index và buộc phải chạy quét toàn bộ bảng (**Sequential Scan**). Khi dữ liệu chạm mốc vài trăm nghìn dòng, CPU sẽ bị vắt kiệt và latency tăng vọt. Hơn nữa, `LIKE` hoàn toàn không có khả năng hiểu ngữ nghĩa, không xử lý được từ gốc (stemming: tìm "chạy" không ra "đang chạy", tìm "run" không ra "running"), và hoàn toàn không thể xếp hạng độ liên quan (Relevance Ranking).

2. **Phương pháp "Dao mổ trâu": Triển khai Elasticsearch / OpenSearch**
   - Cực kỳ mạnh mẽ, hỗ trợ đầy đủ các tính năng tìm kiếm chuyên sâu, fuzzy search, typo-tolerance.
   - **Cái giá phải trả:** Các bạn phải dựng thêm một cụm hạ tầng độc lập (Cluster), tiêu tốn rất nhiều RAM (JVM heap), đau đầu với bài toán đồng bộ dữ liệu thời gian thực giữa cơ sở dữ liệu chính và Elasticsearch (Dual-write dễ sinh bất đồng bộ, phải thiết lập pipeline CDC phức tạp), và tăng gấp đôi gánh nặng bảo trì vận hành (DevOps).

3. **Giải pháp cân bằng hoàn hảo: PostgreSQL Full Text Search (FTS)**
   - Được tích hợp sẵn ngay trong nhân của PostgreSQL mà không cần cài thêm bất kỳ dịch vụ ngoài nào.
   - Dữ liệu được tìm kiếm tức thì theo chuẩn **ACID** (vừa insert/update vào bảng là tìm thấy ngay).
   - Hỗ trợ cấu trúc chỉ mục đảo **GIN Index (Generalized Inverted Index)** cho tốc độ truy vấn chỉ vài mili-giây trên hàng triệu văn bản.
   - Hỗ trợ xếp hạng kết quả theo độ phù hợp bằng hàm `ts_rank`.

Nếu hệ thống của các bạn chưa đạt tới quy mô hàng trăm triệu bản ghi văn bản phức tạp, việc tận dụng tối đa PostgreSQL Full Text Search chính là giải pháp kiến trúc thông minh nhất: Tiết kiệm chi phí, giảm thiểu độ phức tạp của hạ tầng mà vẫn mang lại trải nghiệm tìm kiếm xuất sắc. Hãy cùng mình tìm hiểu nguyên lý và cách cài đặt ngay sau đây!

---

# II. Kiến trúc / Nguyên lý

Để biến PostgreSQL thành một search engine thu nhỏ, chúng ta cần nắm vững 3 thành phần cốt lõi bên dưới kiến trúc của nó:

### 1. Hai kiểu dữ liệu nền tảng: `tsvector` và `tsquery`

PostgreSQL xử lý tìm kiếm văn bản thông qua việc chuyển đổi chuỗi thô thành hai kiểu dữ liệu chuyên biệt:

- **`tsvector` (Text Search Vector):** Là một tập hợp các từ đã được chuẩn hóa (gọi là **lexemes**), loại bỏ các từ dừng vô nghĩa (stopwords như "a", "the", "in", "is"), chuyển các biến thể từ về dạng gốc (Stemming), đồng thời lưu trữ vị trí xuất hiện (positions) của từng từ trong đoạn văn.
  - Ví dụ: Đoạn văn *"The Fat Rats were jumping"* sau khi qua hàm chuẩn hóa sẽ trở thành:
    `'fat':2 'jump':5 'rat':3`
  - Vị trí số `2`, `3`, `5` giúp PostgreSQL tính toán khoảng cách giữa các từ khi người dùng tìm kiếm theo cụm từ liên tiếp (Phrase Search).

- **`tsquery` (Text Search Query):** Đại diện cho các từ khóa tìm kiếm mà người dùng nhập vào, được kết nối với nhau bởi các toán tử logic:
  - `&` (AND): Bắt buộc chứa cả hai từ.
  - `|` (OR): Chứa một trong hai từ.
  - `!` (NOT): Không chứa từ này.
  - `<->` (FOLLOWED BY): Tìm kiếm cụm từ chính xác theo thứ tự liền kề nhau.

Khi tìm kiếm, ta dùng toán tử so khớp **`@@`**:
```sql
SELECT 'fat:2 rat:3'::tsvector @@ 'fat & rat'::tsquery; -- Trả về TRUE
```

### 2. Bộ phân tích ngôn ngữ (Text Search Configuration & Dictionaries)

Làm sao PostgreSQL biết chuyển "jumping" thành "jump" hay "rats" thành "rat"? Đó là nhờ hệ thống cấu hình tìm kiếm văn bản (**Text Search Configuration**), được quản lý trong view hệ thống `pg_ts_config`.

Cơ chế phân tích văn bản bao gồm 3 bước:
1. **Parser (Bộ phân tách):** Tách một chuỗi văn bản dài thành các token thô (từ, số, email, URL, ký tự đặc biệt).
2. **Dictionaries (Từ điển & Bộ lọc):** 
   - Loại bỏ các từ vô nghĩa (**Stopwords**).
   - Áp dụng thuật toán **Stemming** (ví dụ thuật toán Snowball Stemmer cho tiếng Anh) để đưa từ về dạng nguyên thể.
3. **Quoting với Dollar Signs (`$$...$$`):** Trong SQL, khi viết các câu lệnh thử nghiệm với các chuỗi văn bản dài có chứa dấu ngoặc hoặc dấu nháy đơn, cú pháp Dollar-quoting `$$nội dung$$` của PostgreSQL giúp tránh hoàn toàn lỗi escape chuỗi khó chịu.

### 3. GIN Index (Generalized Inverted Index) - Bí mật của tốc độ mili-giây

Tại sao tìm kiếm Full Text trên hàng triệu dòng lại có thể chạy trong chớp mắt? Bí mật nằm ở cấu trúc **GIN Index (Chỉ mục đảo)**.

> **Inverted Index (Chỉ mục đảo)** là kỹ thuật cốt lõi được cả Elasticsearch, Apache Lucene và Google Search sử dụng:
> - Thay vì lập chỉ mục: *Dòng 1 $\rightarrow$ chứa từ A, từ B, từ C*.
> - Inverted Index đảo ngược lại: *Từ A $\rightarrow$ xuất hiện ở Dòng 1, Dòng 45, Dòng 890*.

Khi người dùng tìm kiếm cụm `fat & rat`, PostgreSQL chỉ cần tra cứu cây B-tree của GIN Index để lấy ra danh sách các dòng chứa từ `fat`, lấy danh sách các dòng chứa từ `rat`, rồi thực hiện phép giao (Intersection) giữa hai danh sách này trong bộ nhớ. Toàn bộ quá trình diễn ra mà không cần phải đọc qua bất kỳ nội dung văn bản thô nào trên đĩa!

---

# III. Cài đặt / Hands-on code

Bây giờ, chúng ta sẽ cùng nhau xây dựng một tính năng tìm kiếm sản phẩm hoàn chỉnh từ A đến Z trên PostgreSQL.

### Bước 1: Khám phá các hàm cơ bản với `tsvector` và `tsquery`

Hãy mở terminal hoặc pgAdmin và chạy thử các câu lệnh sau:

```sql
-- 1. Xem cách PostgreSQL chuẩn hóa một câu văn tiếng Anh
SELECT to_tsvector('english', 'The quick brown foxes were jumping over the lazy dog');
-- Kết quả: 'brown':3 'dog':10 'fox':4 'jump':6 'lazi':9 'quick':2

-- 2. Thử nghiệm so khớp với toán tử @@
SELECT to_tsvector('english', 'PostgreSQL provides awesome full text search capabilities') 
    @@ to_tsquery('english', 'awesome & search');
-- Kết quả: true

-- 3. Tìm kiếm cụm từ liền kề bằng toán tử <-> (Phrase search)
SELECT to_tsvector('english', 'PostgreSQL full text search') 
    @@ to_tsquery('english', 'text <-> search');
-- Kết quả: true (vì từ "text" đứng ngay liền trước từ "search")
```

### Bước 2: Thiết kế bảng sản phẩm và cột `search_vector`

Để đạt hiệu năng cao nhất trong production, ta không nên gọi hàm `to_tsvector()` trong mệnh đề `WHERE` ở mỗi câu truy vấn. Thay vào đó, ta tạo sẵn một cột chuyên dụng kiểu `tsvector` và lưu trữ kết quả tính toán sẵn:

```sql
-- Tạo bảng sản phẩm
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    search_vector tsvector,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp()
);

-- Tạo GIN Index trên cột search_vector
CREATE INDEX idx_products_search_vector ON products USING GIN (search_vector);
```

### Bước 3: Tự động cập nhật `search_vector` kèm gán trọng số (Setweight)

Trong một sản phẩm, từ khóa xuất hiện ở **Tiêu đề (Title)** rõ ràng phải có giá trị quan trọng hơn từ khóa nằm trong **Mô tả (Description)**. PostgreSQL hỗ trợ 4 mức trọng số từ cao đến thấp: `A`, `B`, `C`, `D`.

Ta sử dụng trigger để tự động tổng hợp `title` (gán trọng số `A`) và `description` (gán trọng số `B`) vào cột `search_vector` mỗi khi có thao tác INSERT hoặc UPDATE:

```sql
-- Tạo hàm cập nhật search vector có gán trọng số
CREATE OR REPLACE FUNCTION fn_products_search_vector_update()
RETURNS trigger AS $$
BEGIN
    NEW.search_vector := 
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.description, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Gắn trigger vào bảng products
CREATE TRIGGER trg_products_search_vector_update
    BEFORE INSERT OR UPDATE OF title, description ON products
    FOR EACH ROW
    EXECUTE FUNCTION fn_products_search_vector_update();
```

### Bước 4: Chèn dữ liệu mẫu thực nghiệm

```sql
INSERT INTO products (title, description, category) VALUES
('PostgreSQL High Performance Guide', 'Learn how to optimize database queries, build efficient indexes, and tune system memory.', 'Books'),
('Mechanical Keyboard Pro', 'A high performance mechanical keyboard with RGB backlit and hot-swappable switches.', 'Hardware'),
('Full Text Search in Practice', 'Comprehensive tutorial on implementing search engines inside relational databases without external services.', 'Books'),
('Wireless Ergonomic Mouse', 'Comfortable wireless mouse designed for long working hours with high precision sensor.', 'Hardware');
```

Sau khi chèn, cột `search_vector` sẽ được tự động điền đầy đủ các lexemes kèm nhãn trọng số `A` và `B`.

### Bước 5: Viết câu truy vấn tìm kiếm chuyên nghiệp với `plainto_tsquery`, `ts_rank` và Highlight

Khi làm việc với người dùng thực tế, họ sẽ nhập văn bản tự nhiên như *"performance database"* thay vì cú pháp `'performance & database'`. Nếu truyền trực tiếp vào `to_tsquery()`, câu lệnh sẽ báo lỗi cú pháp.

PostgreSQL cung cấp hàm **`plainto_tsquery()`** giúp tự động parse văn bản tự nhiên của người dùng thành `tsquery` hợp lệ. Kết hợp với hàm **`ts_rank()`** để xếp hạng điểm số và **`ts_headline()`** để bôi đậm từ khóa tìm kiếm:

```sql
WITH search_param AS (
    SELECT plainto_tsquery('english', 'performance database') AS query
)
SELECT 
    p.product_id,
    p.title,
    ts_rank(p.search_vector, sp.query) AS relevance_score,
    ts_headline('english', p.description, sp.query, 
        'StartSel = <mark>, StopSel = </mark>, MaxWords=35, MinWords=15'
    ) AS snippet
FROM products p, search_param sp
WHERE p.search_vector @@ sp.query
ORDER BY relevance_score DESC;
```

**Kết quả trả về:**
- Sản phẩm *"PostgreSQL High Performance Guide"* sẽ đứng đầu bảng với `relevance_score` cao nhất (vì từ "Performance" nằm ở tiêu đề mang trọng số `A`, và từ "database" xuất hiện trong mô tả).
- Cột `snippet` trả về đoạn văn bản có gắn sẵn thẻ `<mark>optimize database queries</mark>`, sẵn sàng để render highlight trực tiếp trên giao diện web của các bạn!

### Bước 6: Kiểm tra kế hoạch thực thi (EXPLAIN)

Khi kiểm tra với `EXPLAIN ANALYZE`:
```
Bitmap Heap Scan on products  (cost=8.00..12.50 rows=1 width=300)
  Recheck Cond: (search_vector @@ '''perform'' & ''databas'''::tsquery)
  ->  Bitmap Index Scan on idx_products_search_vector  (cost=0.00..8.00 rows=1 width=0)
        Index Cond: (search_vector @@ '''perform'' & ''databas'''::tsquery)
```
Toàn bộ câu truy vấn được phục vụ hoàn toàn qua **Bitmap Index Scan** trên chỉ mục GIN, đảm bảo độ trễ ổn định dưới vài mili-giây kể cả khi bảng dữ liệu phình to.

---

# IV. Lesson learned / Tổng kết

PostgreSQL một lần nữa khẳng định vị thế "con dao Thụy Sĩ" trong làng cơ sở dữ liệu. Bằng việc tận dụng tính năng Full Text Search có sẵn, các bạn có thể xây dựng một trải nghiệm tìm kiếm mạnh mẽ, chính xác mà không cần đánh đổi bằng sự phức tạp của hạ tầng phân tán.

Để tổng kết lại bài viết này, mình gửi gắm đến các bạn những kinh nghiệm thực chiến sau:

1. **Tuyệt đối tránh `LIKE '%keyword%'` trên tập dữ liệu lớn:** Hãy thay thế ngay bằng Full Text Search để giải phóng CPU của database khỏi thảm họa Seq Scan.
2. **Luôn lưu trữ sẵn `tsvector` và đánh GIN Index:** Không bao giờ tính toán `to_tsvector` trực tiếp trong mệnh đề `WHERE`. Hãy dùng cột lưu trữ riêng, đánh GIN Index và tự động hóa cập nhật qua Trigger hoặc Generated Column.
3. **Phân cấp trọng số với `setweight`:** Tiêu đề, danh mục, từ khóa tóm tắt luôn cần được gắn trọng số `A` hoặc `B`, trong khi nội dung chi tiết mang trọng số `C` hoặc `D` để thuật toán `ts_rank` trả về kết quả chính xác nhất.
4. **Sử dụng `plainto_tsquery` cho User Input:** Tránh lỗi văng Exception khi người dùng vô tình nhập các ký tự đặc biệt (`&`, `|`, `!`).
5. **Biết rõ ranh giới của công cụ:**
   - **Nên dùng PostgreSQL FTS:** Dữ liệu dưới 20-30 triệu dòng, tài nguyên hạ tầng có hạn, yêu cầu cập nhật dữ liệu nhất quán theo thời gian thực (ACID), tìm kiếm văn bản thông thường.
   - **Nên chuyển sang Elasticsearch/Meilisearch:** Khi cần xử lý Big Data hàng trăm triệu dòng, log analytics khổng lồ, hỗ trợ gợi ý từ khóa phức tạp (Autocomplete / Did you mean), hoặc tìm kiếm đa ngôn ngữ với bộ từ điển phân tích từ tùy biến sâu.

Hy vọng bài viết này giúp các bạn tự tin triển khai tính năng Full Text Search ngay trên cụm PostgreSQL sẵn có của mình. Chúc các bạn áp dụng thành công vào các dự án thực tế!
