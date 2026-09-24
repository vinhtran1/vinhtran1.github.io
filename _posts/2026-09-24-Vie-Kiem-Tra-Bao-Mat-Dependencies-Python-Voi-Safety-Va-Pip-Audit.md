---
title: 'Kiểm Tra An Toàn Và Lỗ Hổng Bảo Mật Python Dependencies: Tự Động Hóa Quét requirements.txt Với Safety CLI Và pip-audit'
date: 2026-09-24 07:40:00 +0700
categories: [Security, Python]
tags: [Python, Security, DevSecOps, Vulnerabilities, BestPractices]
keywords: [Python, Security, DevSecOps, Vulnerabilities]
pin: false
image:
  path: /assets/img/posts/2026/kiem-tra-bao-mat-dependencies-python-voi-safety-va-pip-audit/cover.webp
  alt: 'Mô hình DevSecOps tự động hóa quét lỗ hổng bảo mật Python Dependencies bằng Safety CLI và pip-audit tích hợp trong CI/CD'
---

# I. Dẫn nhập: Lý thuyết & Nguyên lý nền tảng

Chào các bạn! Trong thời đại phát triển phần mềm hiện nay, hiếm có ứng dụng nào được viết hoàn toàn từ đầu mà không dựa vào các thư viện bên thứ ba. Theo các thống kê bảo mật mới nhất, mã nguồn của một ứng dụng Python điển hình có tới **80% đến 90%** dung lượng đến từ các thư viện mã nguồn mở tải về từ PyPI (Python Package Index).

Chính sự tiện lợi vượt trội của hệ sinh thái `pip` lại tạo ra một bề mặt tấn công khổng lồ mang tên: **Mối đe dọa Chuỗi cung ứng phần mềm (Software Supply Chain Security)**. Các nhóm tội phạm mạng nhận ra rằng, thay vì tốn công phá vỡ hệ thống tường lửa kiên cố của từng doanh nghiệp, chúng chỉ cần "đầu độc" một gói thư viện phổ biến trên PyPI là có thể âm thầm xâm nhập vào hàng nghìn hệ thống production cùng một lúc.

Các hình thức tấn công nguy hiểm phổ biến trên PyPI bao gồm:
1. **Typosquatting (Đặt tên gây nhầm lẫn)**: Kẻ xấu đăng ký các gói có tên gần giống hệt với thư viện nổi tiếng (ví dụ: `reqeusts` thay vì `requests`, `colourama` thay vì `colorama`). Chỉ một lỗi gõ phím của lập trình viên, mã độc sẽ được tải về, tự động chạy trong script cài đặt (`setup.py`) để đánh cắp SSH keys, file mật khẩu, hoặc biến môi trường `AWS_SECRET_ACCESS_KEY`.
2. **Dependency Confusion (Xung đột gói nội bộ)**: Lợi dụng cơ chế ưu tiên số phiên bản cao hơn của pip để inject một gói độc hại công khai trên PyPI thay thế cho một gói private cùng tên trong mạng nội bộ công ty.
3. **Lỗ hổng bảo mật công bố (Known CVEs)**: Sử dụng các phiên bản thư viện cũ chứa các lỗ hổng thực thi mã từ xa (RCE), SQL Injection, hoặc ReDoS (Regular Expression Denial of Service).
4. **Hiểm họa Phụ thuộc bắc cầu (Transitive Dependencies)**: Bạn chỉ khai báo thư viện `A` trong `requirements.txt`. Nhưng `A` lại phụ thuộc vào `B`, và `B` lại gọi `C`. Lỗ hổng bảo mật nằm sâu ở tầng `C` khiến các kỹ sư kiểm tra thủ công hoàn toàn không thể phát hiện ra.

Để lượng hóa mức độ nghiêm trọng, giới bảo mật sử dụng thang điểm **CVSS v3 (Common Vulnerability Scoring System)** từ 0.0 đến 10.0 (chia thành Low, Medium, High, và Critical). Trong bài viết này, mình sẽ cùng các bạn tìm hiểu phương pháp "Dịch chuyển bảo mật sang trái" (Shift-Left Security), tự động hóa việc quét lỗ hổng phụ thuộc bằng hai công cụ tiêu chuẩn: `Safety CLI` và `pip-audit`.

---

# II. Kiến trúc & So sánh thực tế

### 1. So sánh chuyên sâu: `pip-audit` vs `Safety CLI`

Khi lựa chọn công cụ kiểm định dependencies cho dự án Python, cộng đồng thường cân nhắc giữa `pip-audit` và `Safety CLI`:

```
[Developer Machine]
      │
      ├──> pre-commit hook (Chặn commit nếu file requirements.txt có CVE High/Critical)
      │
[GitLab / GitHub Pull Request]
      │
      ├──> CI Job: pip-audit scanner
      │       │
      │       ├──> [Phát hiện CVE Critical/High] ──> Ghi chú vào PR, Block Merge!
      │       │
      │       └──> [Sạch sẽ / Passed] ──────────> Cho phép chạy Build & Deploy
      │
[Production Container Build]
      │
      └──> Final Runtime Audit trong virtualenv trước khi đóng gói
```

- **pip-audit**:
  - Được phát triển bởi tổ chức an ninh mạng danh tiếng **Trail of Bits** với sự tài trợ chính thức từ OpenSSF (Open Source Security Foundation) và PyPA (Python Packaging Authority).
  - Sử dụng cơ sở dữ liệu mở **OSV (Open Source Vulnerabilities)** do Google bảo trợ kết hợp với PyPA Advisory Database.
  - Hoàn toàn miễn phí, 100% mã nguồn mở, không giới hạn số lượng request, không yêu cầu tài khoản hay API key.
  - Tích hợp tính năng tự động nâng cấp phiên bản không dính lỗi (`pip-audit --fix`).
- **Safety CLI**:
  - Được phát triển bởi SafetyCybersecurity (tiền thân là PyUp).
  - Sử dụng cơ sở dữ liệu Safety DB có đội ngũ chuyên gia nghiên cứu bảo mật kiểm duyệt độc lập.
  - Các phiên bản mới (`safety 3.x`) đã chuyển dần sang mô hình thương mại: yêu cầu đăng ký tài khoản, giới hạn tính năng quét nâng cao cho tài khoản miễn phí.

### 2. Bảng so sánh tính năng toàn diện

| Tiêu chí | `pip-audit` (PyPA / Trail of Bits) | `Safety CLI` (SafetyCybersecurity) |
| :--- | :--- | :--- |
| **Bản quyền & Chi phí** | Apache 2.0 (100% Miễn phí vĩnh viễn) | Freemium / Yêu cầu API Token thương mại |
| **Cơ sở dữ liệu CVE** | OSV.dev & PyPA Advisory Database | Safety DB (Proprietary + Public feeds) |
| **Chế độ quét** | File requirements, lockfiles & runtime venv | File requirements & virtualenv |
| **Tự động vá lỗi (Auto-fix)**| Hỗ trợ mạnh mẽ qua cờ `--fix` | Đưa ra gợi ý nâng cấp phiên bản |
| **Phân tích Transitive** | Quét đệ quy toàn bộ cây phụ thuộc | Quét cây phụ thuộc |
| **Định dạng báo cáo** | SARIF, JSON, Text, Markdown | JSON, HTML, Screen text |

---

# III. Cài đặt & Thực hành code: Triển khai thực chiến

### 1. Thiết lập kịch bản thực nghiệm với các thư viện dính CVE

Để kiểm chứng khả năng phát hiện lỗ hổng, chúng ta tạo một file `vulnerable_requirements.txt` chứa các phiên bản thư viện có CVE nguy hiểm đã được ghi nhận:

```ini
# vulnerable_requirements.txt
# CVE-2021-33503 (CVSS 7.5 - ReDoS dẫn tới sập dịch vụ từ xa)
urllib3==1.26.4

# CVE-2018-18074 (CVSS 7.5 - Rò rỉ thông tin xác thực HTTP Basic Auth khi redirect)
requests==2.20.0

# CVE-2020-28493 (CVSS 7.5 - Arbitrary Code Execution trong Jinja2)
jinja2==2.10.1
```

### 2. Thực thi kiểm định quét lỗ hổng bằng CLI

Các bạn có thể chạy kiểm tra ngay trên máy phát triển hoặc tích hợp vào Makefile:

```bash
# Cài đặt pip-audit
pip install pip-audit

# 1. Quét file requirements trực tiếp với thông tin chi tiết
pip-audit -r vulnerable_requirements.txt --desc on

# 2. Xuất báo cáo định dạng JSON phục vụ xử lý tự động
pip-audit -r vulnerable_requirements.txt -f json -o audit-report.json

# 3. Chạy thử nghiệm cơ chế tự động sửa lỗi (Dry Run)
pip-audit -r vulnerable_requirements.txt --fix --dry-run

# 4. Thực thi tự động nâng cấp phiên bản an toàn vào file requirements
pip-audit -r vulnerable_requirements.txt --fix

# 5. Quét toàn bộ môi trường ảo hiện tại (bao gồm cả transitive dependencies)
pip-audit --local
```

### 3. Tự động hóa kiểm định trong GitHub Actions CI/CD Pipeline

File `.github/workflows/security-audit.yml` dưới đây giúp tự động chặn các pull request nguy hiểm và xuất báo cáo chuẩn SARIF lên tab Security của GitHub:

```yaml
name: Security Audit - Python Dependencies

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    # Quét định kỳ hàng ngày lúc 2h sáng để phát hiện các CVE zero-day mới công bố
    - cron: '0 2 * * *'

jobs:
  dependency-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install Audit Tools & Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pip-audit
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

      - name: Run pip-audit scan on requirements.txt
        run: |
          echo "Bắt đầu quét lỗ hổng phụ thuộc với pip-audit..."
          pip-audit -r requirements.txt --desc on

      - name: Generate SARIF Security Report
        if: always()
        run: |
          pip-audit -r requirements.txt -f sarif -o pip-audit-results.sarif || true

      - name: Upload SARIF to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: pip-audit-results.sarif
```

### 4. Thiết lập Pre-commit Hook gác cổng tại máy lập trình viên

Để ngăn chặn lập trình viên vô tình commit các thư viện dính lỗi lên kho mã nguồn, cấu hình file `.pre-commit-config.yaml`:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pypa/pip-audit
    rev: v2.7.3
    hooks:
      - id: pip-audit
        args: ["-r", "requirements.txt", "--strict"]
        files: ^(requirements.*\.txt|setup\.py|pyproject\.toml)$
```

---

# IV. Lesson learned & Tổng kết: Best Practices & Cạm bẫy production

Khi áp dụng tự động hóa kiểm định dependencies vào các hệ thống lớn, mình xin chia sẻ 4 kinh nghiệm thực chiến đắt giá:

1. **Sai lầm khi chỉ quét file `requirements.txt` cấp cao**:
   - Nếu file `requirements.txt` của các bạn chỉ ghi `requests==2.31.0`, việc chỉ kiểm tra text file sẽ hoàn toàn bỏ qua các thư viện con mà nó cài ngầm (`urllib3`, `certifi`, `idna`).
   - *Khắc phục*: Hãy luôn sử dụng công cụ khóa phiên bản (Lockfile) như `poetry.lock`, `Pipfile.lock` hoặc `pip-compile` (từ `pip-tools`). Hoặc cấu hình lệnh `pip-audit --local` để quét trực tiếp trên môi trường ảo sau khi đã cài đặt đủ các gói.
2. **Xử lý Báo động giả (False Positives) và CVE chưa có bản vá**:
   - Có những CVE điểm thấp (ví dụ CVSS 3.2) xảy ra ở một hàm mà dự án của bạn hoàn toàn không dùng đến, trong khi tác giả thư viện chưa kịp ra bản vá mới.
   - *Giải pháp*: Tuyệt đối không tắt công cụ quét! Hãy sử dụng cờ `--ignore-vuln PYSEC-XXXX-YYYY` kèm theo một ghi chú rõ ràng về **Lý do chấp nhận rủi ro** và **Thời hạn tái kiểm tra (Expiry Date)**.
3. **Chống giả mạo gói với Hash-Checking Mode trong pip**:
   - Kẻ tấn công có thể xâm nhập hệ thống mạng nội bộ để tráo đổi nội dung gói `.whl` nhưng giữ nguyên số phiên bản.
   - *Khắc phục*: Luôn ghim giá trị băm SHA-256 trong file phụ thuộc: `pip install --require-hashes -r requirements.txt`.
4. **Thiết lập lịch quét định kỳ hàng đêm (Nightly Cron Scans)**:
   - Một gói phần mềm hôm nay an toàn tuyệt đối, nhưng ngày mai một lỗ hổng zero-day hoàn toàn có thể được phát hiện. Do đó, việc quét bảo mật không thể chỉ diễn ra khi có commit code, mà bắt buộc phải chạy định kỳ 24h một lần trên nhánh production.

Việc tích hợp `pip-audit` vào quy trình DevSecOps giúp các bạn phát hiện và khắc phục các hiểm họa chuỗi cung ứng từ sớm, tiết kiệm hàng trăm giờ cứu hỏa sự cố và bảo vệ vững chắc dữ liệu người dùng!
