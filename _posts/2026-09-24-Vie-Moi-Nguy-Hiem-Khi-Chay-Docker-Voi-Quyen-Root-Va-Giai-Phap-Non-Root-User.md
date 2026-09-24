---
title: 'Mối Nguy Hiểm Khi Chạy Docker Với Quyen Root Và Chiến Lược Bảo Mật Container Bằng Non-Root User & User Namespace'
date: 2026-09-24 07:30:00 +0700
categories: [DevOps, Security]
tags: [Docker, Security, Containers, Linux, BestPractices]
keywords: [Docker, Security, Containers, Linux]
pin: false
image:
  path: /assets/img/posts/2026/moi-nguy-hiem-khi-chay-docker-voi-quyen-root-va-giai-phap-non-root-user/cover.webp
  alt: 'Kiến trúc bảo mật Container: Nguy cơ Container Breakout với UID 0 và giải pháp Non-Root User cùng Linux User Namespaces'
---

# I. Dẫn nhập: Lý thuyết & Nguyên lý nền tảng

Chào các bạn! Khi mới bước chân vào thế giới DevOps hay containerization, hầu hết chúng ta đều có một cảm giác rất an tâm khi đóng gói ứng dụng vào Docker. Nhiều bạn vẫn giữ một quan niệm sai lầm phổ biến: coi container như một máy ảo (Virtual Machine - VM), một thế giới biệt lập hoàn toàn có hệ điều hành riêng, kernel riêng và được bảo vệ bởi lớp ảo hóa phần cứng (hypervisor).

Nhưng sự thật thì không phải như vậy. Container **không phải là máy ảo**. Về bản chất, container thực chất chỉ là một tiến trình thông thường (isolated process) chạy trực tiếp trên nhân Linux (Linux Kernel) của máy chủ Host, được bảo bọc bởi ba trụ cột cốt lõi của hệ điều hành Linux:

1. **Linux Namespaces** (PID, NET, MNT, IPC, UTS, USER): Tạo ra các góc nhìn ảo hóa độc lập về tài nguyên hệ thống (tiến trình, mạng, file system...).
2. **Control Groups (cgroups)**: Đo lường, giới hạn và phân bổ tài nguyên phần cứng (CPU quota, RAM limits, Disk I/O).
3. **Linux Capabilities & Seccomp Profiles**: Thu hẹp và giới hạn danh sách các system calls (lời gọi hệ thống) mà tiến trình được phép yêu cầu kernel thực thi.

Chính vì container chia sẻ chung Linux Kernel với máy chủ Host, một vấn đề bảo mật nghiêm trọng xuất hiện: **Nếu trong Dockerfile các bạn không khai báo chỉ thị `USER`, tiến trình bên trong container sẽ mặc định khởi chạy dưới quyền `root` (UID 0)**.

Điều nguy hiểm tột cùng ở đây là gì? Mặc định, **UID 0 bên trong container chính là UID 0 (root) trên máy chủ Host!** Kernel Linux không phân biệt UID 0 đó được gọi từ namespace nào nếu bạn không kích hoạt cơ chế cách ly user namespace. Nếu ứng dụng của các bạn dính một lỗ hổng thực thi mã từ xa (Remote Code Execution - RCE), kẻ tấn công ngay lập tức có được quyền root trong container. Và từ đó, khoảng cách để chúng thoát khỏi chiếc lồng container (Container Breakout / Escape) và chiếm quyền điều khiển hoàn toàn máy chủ production chỉ còn là một bước ngắn.

Trong bài viết này, mình sẽ cùng các bạn phân tích bản chất cơ chế cô lập của container, mổ xẻ các vector tấn công Container Escape kinh điển và thiết lập chiến lược phòng thủ vững chắc bằng việc áp dụng Non-Root User kết hợp với Linux User Namespaces (`userns-remap`).

---

# II. Kiến trúc & So sánh thực tế

### 1. Mổ xẻ các Vector tấn công Container Breakout khi chạy Root

Khi một tiến trình container chạy với UID 0, kẻ tấn công có thể khai thác nhiều điểm yếu chí mạng trong hạ tầng để leo thang đặc quyền ra máy chủ Host:

```
[TRƯỜNG HỢP NGUY HIỂM: Default Root]
Container Process (UID 0: root) ════(Container Breakout)════> Host Kernel (UID 0: ROOT TOÀN HỆ THỐNG!)
                                                                 │
                                                                 ▼ [Thảm họa: Chiếm quyền kiểm soát Host]

[TRƯỜNG HỢP AN TOÀN: Non-Root + User Namespace Remap]
Container Process (UID 10001: appuser) ────> Host OS (UID 10001: Unprivileged user)
        hoặc
Container Process (UID 0 trong userns) ────> Host OS (UID 165536: SubUID hoàn toàn vô hại!)
                                                    │
                                                    ▼ [Container Breakout bị chặn đứng bởi Kernel DAC]
```

1. **Hiểm họa Docker Socket Mount (`/var/run/docker.sock`)**:
   - Đây là cạm bẫy phổ biến nhất mà các kỹ sư hay mắc phải khi cấu hình CI/CD runner (Docker-in-Docker) hoặc các công cụ monitoring.
   - Khi mount `/var/run/docker.sock` vào container chạy quyền root, kẻ tấn công bên trong container chỉ cần gửi một lệnh HTTP API tới Docker Daemon trên Host để tạo một container mới với cờ `--privileged --net=host -v /:/host-root`. Lúc này, toàn bộ file system của Host OS được mount vào và kẻ tấn công có quyền ghi đè `/etc/shadow`, cài đặt rootkit chỉ trong vòng 5 giây!
2. **Khai thác lỗ hổng Container Runtime (CVE-2019-5736 runc Breakout)**:
   - Khi container chạy UID 0, kẻ tấn công có thể ghi đè binary của chính `runc` trên Host thông qua file descriptor `/proc/self/exe` khi một admin chạy `docker exec` vào container bị xâm nhập. Khi `runc` bị nhiễm mã độc, bất kỳ container nào khởi chạy tiếp theo đều sẽ thực thi mã độc với quyền root máy chủ Host.
3. **Khai thác lỗ hổng Linux Kernel (Dirty COW, Dirty Pipe)**:
   - Các lỗ hổng leo thang đặc quyền bộ nhớ trong Linux Kernel (như CVE-2016-5195 Dirty COW hay CVE-2022-0847 Dirty Pipe) cho phép tiến trình ghi đè vào các trang nhớ chỉ đọc (read-only memory page cache). Nếu tiến trình đã sẵn quyền UID 0 trong container, việc vượt qua DAC (Discretionary Access Control) của kernel để thâm nhập Host trở nên đơn giản hơn rất nhiều.

### 2. Hai lớp phòng thủ: Non-Root User và User Namespaces (`userns-remap`)

Để hóa giải hoàn toàn các mối nguy trên, chúng ta cần triển khai mô hình bảo mật hai lớp chuyên sâu:

- **Lớp 1: Ép buộc Non-Root User trong Dockerfile & Container Runtime**:
  - Khởi tạo một user/group riêng biệt không có đặc quyền (ví dụ UID 10001 / GID 10001), không cấp quyền `sudo`, và vô hiệu hóa login shell (`/sbin/nologin`).
  - Toàn bộ source code và dependencies chỉ được phân quyền tối thiểu (read-only đối với mã nguồn, chỉ ghi vào thư mục tạm thời được kiểm soát).
- **Lớp 2: Kích hoạt Linux User Namespace Remapping (`userns-remap`)**:
  - User Namespaces cho phép Linux Kernel ánh xạ một dải UID/GID bên trong container sang một dải UID/GID hoàn toàn khác trên Host OS.
  - Cụ thể: UID 0 bên trong container sẽ được kernel ánh xạ sang UID 100000 hoặc 165536 trên Host. Cho dù ứng dụng có bị chiếm root ảo trong container, đối với hệ điều hành Host, tiến trình đó chỉ là một user vô danh không có quyền can thiệp vào bất kỳ file hệ thống nào (`/etc`, `/bin`, `/var`).

### 3. Bảng so sánh đa chiều giữa các phương thức vận hành

| Tiêu chí | Root Container (Mặc định) | Non-Root User (`USER 10001`) | User Namespace Remap (`userns-remap`) |
| :--- | :--- | :--- | :--- |
| **UID bên trong container** | UID 0 (root) | UID 10001 (appuser) | UID 0 (root ảo) |
| **UID thực tế trên Host OS** | UID 0 (root thật!) | UID 10001 (không đặc quyền) | UID 165536+ (SubUID cô lập) |
| **Khả năng leo thang khi Breakout** | Toàn quyền kiểm soát Host | Bị chặn bởi DAC Host | Bị chặn hoàn toàn bởi Kernel |
| **Khả năng mở cổng mạng < 1024** | Cho phép tự do | Bị chặn (trừ khi dùng `cap_net_bind_service`) | Cho phép trong container netns |
| **Độ phức tạp cấu hình** | Không cần làm gì (0 cấu hình) | Rất đơn giản trong Dockerfile | Cần cấu hình daemon.json máy chủ |
| **Tác động tới Volume Storage** | Không bị permission denied | Cần chown thư mục volume | Cần map quyền storage host với subuid |

---

# III. Cài đặt & Thực hành code: Triển khai thực chiến

### 1. Xây dựng Dockerfile chuẩn Non-Root Production (Multi-stage Build)

Dưới đây là một Dockerfile mẫu chuẩn Enterprise cho ứng dụng Python FastAPI / Flask áp dụng multi-stage build, tách biệt hoàn toàn môi trường build và runtime non-root:

```dockerfile
# Stage 1: Build dependencies trong môi trường builder
FROM python:3.11-slim-bullseye AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Runtime image tối giản, bảo mật cao
FROM python:3.11-slim-bullseye AS production

# Thiết lập các biến môi trường chuẩn hóa
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    APP_HOME=/app \
    APP_USER=apprunner \
    APP_UID=10001 \
    APP_GID=10001

WORKDIR $APP_HOME

# 1. Khởi tạo Group và User non-root với UID/GID cố định, không tạo home và khóa login shell
RUN groupadd --gid $APP_GID $APP_USER && \
    useradd --uid $APP_UID --gid $APP_GID --no-create-home \
            --shell /sbin/nologin --comment "Dedicated Service Account" $APP_USER

# 2. Sao chép thư viện đã biên dịch từ builder sang thư mục người dùng
COPY --from=builder /root/.local /home/$APP_USER/.local
COPY --chown=$APP_UID:$APP_GID . $APP_HOME

# 3. Phân quyền chặt chẽ: mã nguồn chỉ đọc (550), chỉ mở quyền ghi cho thư mục logs/tmp (770)
RUN mkdir -p $APP_HOME/logs $APP_HOME/tmp && \
    chown -R $APP_UID:$APP_GID $APP_HOME && \
    chmod -R 550 $APP_HOME && \
    chmod -R 770 $APP_HOME/logs $APP_HOME/tmp

# 4. Cấu hình biến môi trường PATH để nạp các package python
ENV PATH=/home/$APP_USER/.local/bin:$PATH

# 5. CHUYỂN SANG USER NON-ROOT (Bắt buộc dùng số nguyên UID:GID)
USER $APP_UID:$APP_GID

# 6. Expose cổng mạng không đặc quyền (> 1024)
EXPOSE 8080

ENTRYPOINT ["python", "app.py"]
```

### 2. Kích hoạt User Namespace Remapping trên Docker Daemon Host

Để bật tính năng `userns-remap` trên máy chủ Ubuntu/Debian, các bạn thực hiện cấu hình `/etc/docker/daemon.json`:

```bash
# Kiểm tra sự tồn tại của file /etc/subuid và /etc/subgid
sudo touch /etc/subuid /etc/subgid

# Khởi tạo dải subuid cho user dockremap (65536 UIDs bắt đầu từ 100000)
sudo usermod -v 100000-165535 -w 100000-165535 dockremap || echo "dockremap:100000:65536" | sudo tee -a /etc/subuid
echo "dockremap:100000:65536" | sudo tee -a /etc/subgid

# Cấu hình daemon.json với các tham số bảo vệ tối cao
sudo tee /etc/docker/daemon.json <<EOF
{
  "userns-remap": "default",
  "no-new-privileges": true,
  "live-restore": true
}
EOF

# Khởi động lại Docker daemon để nạp cấu hình mới
sudo systemctl restart docker
```

### 3. Khóa cứng Container qua Docker Compose & Kubernetes SecurityContext

Khi triển khai trên môi trường Docker Compose hoặc Kubernetes Pods, chúng ta áp dụng cơ chế Least Privilege (Đặc quyền tối thiểu):

```yaml
# docker-compose.prod.yml
version: '3.8'
services:
  secure-api:
    build: .
    user: "10001:10001"
    read_only: true              # Khóa file system gốc thành read-only
    cap_drop:
      - ALL                      # Tước bỏ 100% Linux capabilities
    cap_add:
      - NET_BIND_SERVICE         # Chỉ cấp lại quyền bind cổng nếu thực sự cần
    security_opt:
      - no-new-privileges:true   # Chặn leo thang đặc quyền qua setuid binary
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
    ports:
      - "8080:8080"
```

```yaml
# Kubernetes Pod SecurityContext tiêu chuẩn sản xuất
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hardened-service
spec:
  replicas: 3
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: registry.internal/app:v1.0.0
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
              - ALL
```

---

# IV. Lesson learned & Tổng kết: Best Practices & Cạm bẫy production

Sau nhiều năm hỗ trợ các đội ngũ vận hành hệ thống container trong sản xuất, mình đã đúc kết được 3 cạm bẫy lớn nhất mà các kỹ sư thường gặp khi chuyển đổi từ root sang non-root:

1. **Lỗi `Permission Denied` khi Mount Host Volume hoặc Persistent Volume (PV/PVC)**:
   - *Triệu chứng*: Khi mount một thư mục từ Host vào container, ứng dụng lập tức crash vì không thể ghi log hay ghi file tạm.
   - *Nguyên nhân*: Thư mục trên Host thuộc sở hữu của `root:root` (chmod 755), trong khi tiến trình container chạy với UID 10001.
   - *Giải pháp*: Trong Docker Compose, bạn cần `chown -R 10001:10001 /data/path` trên Host trước. Trong Kubernetes, hãy tận dụng `securityContext.fsGroup: 10001` hoặc sử dụng một `initContainers` ngắn chạy dưới quyền root chỉ để thực hiện `chown -R 10001:10001 /mount-dir` trước khi container chính khởi chạy.
2. **Lỗi không bind được cổng mạng chuẩn (Port 80, 443)**:
   - *Triệu chứng*: Ứng dụng báo lỗi `bind: permission denied` khi cố gắng lắng nghe trên cổng 80.
   - *Nguyên nhân*: Nhân Linux quy định tất cả các cổng mạng nhỏ hơn 1024 là "Privileged Ports", chỉ tiến trình có capability `CAP_NET_BIND_SERVICE` hoặc chạy UID 0 mới được mở.
   - *Giải pháp*: Thiết kế ứng dụng lắng nghe trên các cổng không đặc quyền như `8080` hoặc `8443`. Hãy để việc lắng nghe cổng 80/443 cho lớp Nginx Reverse Proxy, Ingress Controller hoặc AWS ALB đảm nhiệm.
3. **Cạm bẫy khai báo User bằng Tên thay vì Số (`USER appuser` vs `USER 10001`)**:
   - *Rủi ro*: Nếu các bạn khai báo `USER appuser`, Docker daemon phải dựa vào file `/etc/passwd` bên trong container image để phân giải UID. Nếu hacker can thiệp được vào một base image ở layer thấp và đổi UID của `appuser` thành `0`, bạn sẽ vô tình chạy root mà không hề hay biết!
   - *Quy tắc vàng*: Luôn luôn khai báo định danh user bằng số nguyên: `USER 10001:10001`.

Tóm lại, bảo mật container không phải là một tính năng bổ sung có thể làm sau, mà phải là tư duy thiết kế cốt lõi (Security by Design). Chỉ với vài chỉ thị đơn giản trong Dockerfile và cấu hình daemon, các bạn đã có thể triệt tiêu tới 90% nguy cơ Container Breakout trên hệ thống của mình!
