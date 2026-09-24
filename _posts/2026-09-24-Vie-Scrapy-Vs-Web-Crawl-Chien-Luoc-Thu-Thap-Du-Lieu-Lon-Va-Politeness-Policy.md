---
title: 'Scrapy vs Large-scale Web Crawling: Kiến Trúc Thu Thập Dữ Liệu Phân Tán, Tối Ưu Middleware, robots.txt Và Politeness Policy'
date: 2026-09-24 07:45:00 +0700
categories: [DataEngineering, Python]
tags: [Python, Scrapy, WebScraping, DataEngineering, DistributedSystems]
keywords: [Python, Scrapy, WebScraping, DataEngineering]
pin: false
image:
  path: /assets/img/posts/2026/scrapy-vs-web-crawl-chien-luoc-thu-thap-du-lieu-lon-va-politeness-policy/cover.webp
  alt: 'Kiến trúc Scrapy Engine bất đồng bộ với Twisted, Downloader Middleware, cơ chế điều tiết tự động AutoThrottle và tuân thủ robots.txt'
---

# I. Dẫn nhập: Lý thuyết & Nguyên lý nền tảng

Chào các bạn! Trong kỷ nguyên dữ liệu lớn (Big Data) và trí tuệ nhân tạo (AI/LLM), nhu cầu thu thập tri thức từ mạng Internet ngày càng bùng nổ. Tuy nhiên, giữa việc viết một script nhỏ để cào vài trăm trang tin tức và việc vận hành một hệ thống crawl hàng chục triệu trang web mỗi ngày là một khoảng cách khổng lồ về mặt kỹ thuật hạ tầng.

Trước hết, chúng ta cần phân biệt rạch ròi hai khái niệm thường bị đánh đồng:
- **Web Scraping (Bóc tách dữ liệu có chủ đích)**: Thường nhắm vào một danh sách URL xác định từ trước, với cấu trúc HTML đã biết (ví dụ: bóc tách giá 1.000 sản phẩm cụ thể trên một trang thương mại điện tử).
- **Web Crawling (Duyệt đồ thị Web quy mô lớn)**: Bắt đầu từ một tập hợp URL hạt giống (Seed URLs), crawler tự động tải trang, phân tích ngữ nghĩa, trích xuất tất cả các liên kết mới (`<a href="...">`), đưa chúng vào hàng đợi khám phá (URL Frontier) và tiếp tục duyệt đệ quy để lập bản đồ hàng triệu trang web.

Khi các bạn mở rộng quy mô từ 1.000 trang lên **10.000.000 trang**, một script đơn giản dùng `requests + BeautifulSoup` sẽ lập tức sụp đổ vì hàng loạt giới hạn vật lý:
1. **Nghẽn Network I/O & Chi phí luồng (Thread Overhead)**: Thư viện `requests` hoạt động đồng bộ (Synchronous Blocking). CPU máy chủ nhàn rỗi tới 99% thời gian để chờ phản hồi từ socket mạng. Nếu các bạn cố tạo ra 1.000 threads để tăng tốc, hệ thống sẽ cạn kiệt RAM và sập vì chi phí chuyển ngữ cảnh (Context Switching).
2. **Nổ tung bộ nhớ URL Frontier (Memory Explosion)**: Lưu trữ hàng chục triệu URL chưa thăm trong RAM sẽ dẫn tới lỗi Out-Of-Memory (OOM).
3. **Bị chặn IP và Thử thách WAF (Cloudflare, Akamai)**: Bắn request với tốc độ quá nhanh sẽ khiến tường lửa chặn toàn bộ dải IP của bạn.
4. **Cơn bão phân giải tên miền (DNS Storm)**: Hàng nghìn kết nối đồng thời làm tê liệt bộ nhớ cache DNS nội bộ.

Chính vì vậy, một hệ thống thu thập dữ liệu chuyên nghiệp bắt buộc phải áp dụng **Chính sách lịch sự (Politeness Policy)**. Thu thập dữ liệu không phải là tấn công từ chối dịch vụ (DDoS)! Một crawler chuẩn mực phải biết tự điều tiết tốc độ, tôn trọng tệp `robots.txt` của website đích và phân bổ tải hợp lý. Trong bài viết này, mình sẽ cùng các bạn mổ xẻ kiến trúc bất đồng bộ của Scrapy và xây dựng pipeline thu thập dữ liệu phân tán bền bỉ.

---

# II. Kiến trúc & So sánh thực tế

### 1. Giải phẫu kiến trúc luồng dữ liệu nội bộ của Scrapy

Trái tim của Scrapy là framework mạng bất đồng bộ **Twisted** (hoạt động dựa trên Reactor Pattern). Luồng xử lý dữ liệu qua 8 bước tuần hoàn như sau:

```
┌────────────────────────────────────────────────────────────────────────┐
│                             SCRAPY ENGINE                              │
│                                                                        │
│   ┌───────────────┐     Requests     ┌───────────────┐                 │
│   │               │ ───────────────> │               │                 │
│   │    SPIDERS    │                  │   SCHEDULER   │ (URL Frontier   │
│   │               │ <─────────────── │               │  LIFO/FIFO/BFO) │
│   └───────┬───────┘   Next Request   └───────────────┘                 │
│           │                                                            │
│     Items │                                                            │
│           ▼                                                            │
│   ┌───────────────┐                  ┌───────────────┐                 │
│   │ ITEM PIPELINE │                  │  DOWNLOADER   │ ──(HTTP/HTTPS)──> [INTERNET]
│   └───────────────┘                  └───────┬───────┘                 │
│                                              │ Responses               │
│                                              ▼                         │
│                                     [Downloader Middlewares]           │
│                                     - User-Agent Rotator               │
│                                     - Proxy Pool Rotator               │
│                                     - Retry / AutoThrottle             │
└────────────────────────────────────────────────────────────────────────┘
```

1. **Spider** khởi tạo các Request ban đầu và chuyển tới **Scrapy Engine**.
2. **Engine** đẩy Request vào **Scheduler** để sắp xếp thứ tự ưu tiên (URL Frontier).
3. **Scheduler** phản hồi Request tiếp theo cần xử lý cho Engine.
4. **Engine** chuyển tiếp Request qua chuỗi **Downloader Middlewares** (nơi can thiệp proxy, headers, cookies) để gửi tới **Downloader**.
5. **Downloader** tải trang web bất đồng bộ qua mạng và đóng gói thành đối tượng `Response`, chuyển ngược về Engine.
6. **Engine** đưa Response qua **Spider Middlewares** vào phương thức `parse()` của **Spider**.
7. **Spider** phân tích cấu trúc DOM, bóc tách ra các thực thể dữ liệu (**Item**) và các liên kết mới (**New Requests**).
8. **Engine** chuyển Item tới **Item Pipeline** để làm sạch và lưu trữ (Database, Parquet), đồng thời đưa các New Requests trở lại Scheduler.

### 2. So sánh các mô hình kiến trúc thu thập dữ liệu

| Tiêu chí | `requests + BeautifulSoup` | `Scrapy Standalone` | `Scrapy-Redis Distributed` |
| :--- | :--- | :--- | :--- |
| **Mô hình I/O** | Synchronous Blocking | Asynchronous Non-blocking (Twisted) | Asynchronous Phân tán đa máy chủ |
| **Xử lý URL Frontier** | Tự quản lý (list/set trong RAM) | Built-in Scheduler (Disk/Memory queue) | Redis Sorted Set / Priority Queue |
| **Lọc trùng lặp (Dedup)** | Memory `set()` (Dễ crash OOM) | RFPDupeFilter (SHA1 fingerprint) | Redis Set hoặc Scalable Bloom Filter |
| **Khả năng mở rộng** | Rất thấp, giới hạn 1 máy | Rất cao trên 1 máy đơn | Mở rộng vô hạn theo chiều ngang |
| **Cơ chế Politeness** | Viết sleep thủ công | AutoThrottle Extension chuẩn mực | AutoThrottle phân tán qua Redis delay |
| **Phù hợp cho** | Script nhanh < 5.000 trang | Cào 50.000 - 5.000.000 trang | Cào dữ liệu lớn (> 100.000.000 trang) |

---

# III. Cài đặt & Thực hành code: Triển khai thực chiến

### 1. Xây dựng Spider e-commerce với Politeness Policy và AutoThrottle

Dưới đây là một Spider hoàn chỉnh tuân thủ tuyệt đối quy chuẩn đạo đức `robots.txt` và sử dụng thuật toán AutoThrottle để tự động điều chỉnh tốc độ tải dựa trên độ trễ phản hồi của máy chủ đích:

```python
# ecommerce_crawler/spiders/catalog_spider.py
import scrapy
from urllib.parse import urljoin

class CatalogSpider(scrapy.Spider):
    name = "catalog_spider"
    allowed_domains = ["sandbox-books.toscrape.com"]
    start_urls = ["https://sandbox-books.toscrape.com/index.html"]

    custom_settings = {
        # 1. Tuân thủ chỉ thị robots.txt
        "ROBOTSTXT_OBEY": True,
        
        # 2. Kích hoạt thuật toán AutoThrottle thích ứng độ trễ mạng
        "AUTOTHROTTLE_ENABLED": True,
        "AUTOTHROTTLE_START_DELAY": 1.0,        # Độ trễ khởi đầu (giây)
        "AUTOTHROTTLE_MAX_DELAY": 10.0,         # Độ trễ trần khi máy chủ quá tải
        "AUTOTHROTTLE_TARGET_CONCURRENCY": 2.0, # Trung bình 2 request đồng thời trên 1 domain
        "AUTOTHROTTLE_DEBUG": True,
        
        # 3. Giới hạn số lượng kết nối đồng thời bảo vệ hạ tầng máy chủ
        "CONCURRENT_REQUESTS_PER_DOMAIN": 4,
        "DOWNLOAD_TIMEOUT": 15,
        "COOKIES_ENABLED": False,              # Tắt cookies nếu không cần xác thực để tăng tốc
    }

    def parse(self, response):
        # Trích xuất dữ liệu sản phẩm trên trang hiện tại
        for book in response.css("article.product_pod"):
            yield {
                "title": book.css("h3 a::attr(title)").get(),
                "price": book.css("p.price_color::text").get(),
                "availability": book.css("p.instock.availability::text").re_first(r"\S+"),
                "url": urljoin(response.url, book.css("h3 a::attr(href)").get()),
            }

        # Khám phá liên kết phân trang tiếp theo (Pagination Link Crawling)
        next_page = response.css("li.next a::attr(href)").get()
        if next_page is not None:
            yield response.follow(next_page, callback=self.parse)
```

### 2. Viết Custom Downloader Middleware xoay tua User-Agent và Proxy Pool

Middleware dưới đây giúp luân chuyển ngẫu nhiên User-Agent và IP proxy cho từng request, đồng thời bắt các tín hiệu phản hồi HTTP 429 để cảnh báo:

```python
# ecommerce_crawler/middlewares.py
import random
import logging

class RotateUserAgentAndProxyMiddleware:
    """Downloader Middleware tự động xoay tua danh tính và IP an toàn."""

    def __init__(self, user_agents, proxies):
        self.user_agents = user_agents
        self.proxies = proxies
        self.logger = logging.getLogger(__name__)

    @classmethod
    def from_crawler(cls, crawler):
        user_agents = crawler.settings.getlist("USER_AGENT_LIST") or [
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
            "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36",
            "Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/121.0"
        ]
        proxies = crawler.settings.getlist("PROXY_POOL_LIST") or []
        return cls(user_agents, proxies)

    def process_request(self, request, spider):
        # Gán User-Agent ngẫu nhiên cho từng request
        ua = random.choice(self.user_agents)
        request.headers["User-Agent"] = ua

        # Gán Proxy ngẫu nhiên nếu có danh sách proxy hợp lệ
        if self.proxies and "dont_proxy" not in request.meta:
            proxy = random.choice(self.proxies)
            request.meta["proxy"] = proxy
            self.logger.debug(f"Đang route request qua proxy: {proxy}")

    def process_response(self, request, response, spider):
        # Bắt mã trạng thái giới hạn tốc độ (HTTP 429 Too Many Requests hoặc 403 Forbidden)
        if response.status in [429, 403]:
            self.logger.warning(f"Cảnh báo rate limit tại {response.url} (Status: {response.status})! Kích hoạt cơ chế giảm tốc.")
        return response
```

### 3. Item Pipeline ghi dữ liệu Streaming sang định dạng Parquet với Batch Flushing

Để tối ưu hóa việc lưu trữ hàng triệu bản ghi mà không gây nghẽn I/O đĩa cứng, pipeline này gom dữ liệu vào buffer bộ nhớ và ghi định kỳ ra file Parquet dạng nén Snappy:

```python
# ecommerce_crawler/pipelines.py
import pyarrow as pa
import pyarrow.parquet as pq

class ParquetStoragePipeline:
    """Pipeline gom nhóm dữ liệu và ghi streaming sang file Parquet tối ưu dung lượng."""

    def __init__(self, batch_size=1000, output_file="crawled_data.parquet"):
        self.batch_size = batch_size
        self.output_file = output_file
        self.buffer = []
        self.writer = None
        self.schema = pa.schema([
            ("title", pa.string()),
            ("price", pa.string()),
            ("availability", pa.string()),
            ("url", pa.string()),
        ])

    def open_spider(self, spider):
        self.writer = pq.ParquetWriter(self.output_file, self.schema, compression="snappy")

    def process_item(self, item, spider):
        self.buffer.append(dict(item))
        if len(self.buffer) >= self.batch_size:
            self._flush_batch()
        return item

    def _flush_batch(self):
        if not self.buffer:
            return
        table = pa.Table.from_pylist(self.buffer, schema=self.schema)
        self.writer.write_table(table)
        self.buffer.clear()

    def close_spider(self, spider):
        self._flush_batch()
        if self.writer:
            self.writer.close()
```

---

# IV. Lesson learned & Tổng kết: Best Practices & Cạm bẫy production

Trong quá trình xây dựng các pipeline crawler quy mô hàng trăm triệu bản ghi, mình nhận thấy 4 cạm bẫy "chết người" mà các bạn cần đặc biệt phòng tránh:

1. **Bẫy nhện (Spider Traps) và Vòng lặp vô tận**:
   - Nhiều trang web có tiện ích lịch (Calendar Widget) với URL dạng `/events?year=2026&month=1`. Mỗi lần spider click nút "Next Month", website lại sinh ra một URL mới vô hạn cho tới năm 9999!
   - *Khắc phục*: Luôn giới hạn độ sâu thu thập bằng `DEPTH_LIMIT = 5` trong Scrapy settings, chuẩn hóa URL loại bỏ query parameters thừa và lọc blacklist URL bằng biểu thức chính quy Regex.
2. **Rò rỉ bộ nhớ (Memory Leak) do tham chiếu Response**:
   - Đối tượng `response` trong Scrapy lưu toàn bộ cây DOM và text HTML. Nếu các bạn vô tình gán `response` vào một biến toàn cục hoặc đóng gói trong một callback closure dài hạn, bộ nhớ RAM sẽ tăng vọt lên hàng chục GB và dẫn tới sập container.
   - *Quy tắc*: Chỉ trích xuất chuỗi string cụ thể cần lưu trữ và để bộ thu gom rác (Garbage Collector) của Python giải phóng hoàn toàn response ngay khi rời khỏi hàm `parse()`.
3. **Hiểm họa Zip Bomb và phản hồi dung lượng vô hạn**:
   - Một số máy chủ có thể phản hồi một file nén Gzip 10MB nhưng khi giải nén ra tới 100GB dữ liệu rác nhằm làm tê liệt crawler.
   - *Khắc phục*: Thiết lập trần dung lượng tải về `DOWNLOAD_MAXSIZE = 10485760` (10MB) trong cấu hình Scrapy.
4. **Đạo đức và Pháp lý (Ethical & Legal Compliance)**:
   - Hãy luôn là một công dân mạng văn minh: không cào dữ liệu thông tin cá nhân định danh (PII), tôn trọng điều khoản sử dụng (Terms of Service) và luôn khai báo thông tin liên hệ minh bạch trong HTTP User-Agent: `User-Agent: MyResearchBot/1.0 (+https://mycompany.com/bot-info)`.

Việc làm chủ Scrapy cùng các nguyên tắc Politeness Policy sẽ giúp các bạn tự tin thiết kế những hệ thống thu thập dữ liệu lớn vận hành bền bỉ, hiệu quả và an toàn trong môi trường sản xuất!
