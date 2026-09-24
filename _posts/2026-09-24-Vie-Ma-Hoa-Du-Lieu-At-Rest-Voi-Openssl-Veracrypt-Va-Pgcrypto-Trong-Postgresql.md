---
title: 'Mã Hóa Dữ Liệu Tại Chỗ (Data at Rest): Chiến Lược Phòng Thủ Đa Tầng Với OpenSSL Envelope, VeraCrypt Volume Và PostgreSQL pgcrypto'
date: 2026-09-24 07:50:00 +0700
categories: [Security, Database]
tags: [Security, Cryptography, PostgreSQL, Linux, Encryption]
keywords: [Security, Cryptography, PostgreSQL, Encryption]
pin: false
image:
  path: /assets/img/posts/2026/ma-hoa-du-lieu-at-rest-voi-openssl-veracrypt-va-pgcrypto-trong-postgresql/cover.webp
  alt: 'Chiến lược phòng thủ đa tầng bảo vệ dữ liệu Data-at-Rest: Mã hóa tập tin OpenSSL Envelope, phân vùng VeraCrypt và mã hóa cột cơ sở dữ liệu với PostgreSQL pgcrypto'
---

# I. Dẫn nhập: Lý thuyết & Nguyên lý nền tảng

Chào các bạn! Trong lĩnh vực an toàn thông tin và kiến trúc dữ liệu, mọi dòng dữ liệu phát sinh trong doanh nghiệp đều luân chuyển qua ba trạng thái cơ bản (Three States of Digital Data):

1. **Data in Transit (Dữ liệu khi truyền tải)**: Dữ liệu đang di chuyển qua mạng cáp quang, Internet hoặc giữa các microservices. Trạng thái này hầu như đã được bảo vệ rất tốt bằng TLS 1.3, HTTPS, gRPC mã hóa và mTLS.
2. **Data in Use (Dữ liệu khi đang xử lý)**: Dữ liệu đang được nạp vào thanh ghi CPU và RAM (hiện đang được tăng cường phòng thủ bằng các công nghệ như Confidential Computing, Intel SGX, AMD SEV).
3. **Data at Rest (Dữ liệu tại chỗ / lưu trữ tĩnh)**: Dữ liệu được ghi cố định vào ổ cứng vật lý (NVMe SSD, HDD), mạng lưu trữ SAN, snapshot ổ đĩa ảo (AWS EBS, GCP Persistent Disk), các file sao lưu định kỳ (`backup.sql.gz`) và từng bảng dữ liệu trong hệ quản trị cơ sở dữ liệu.

Tại sao **Data at Rest** lại luôn là mục tiêu săn đón hàng đầu của các cuộc tấn công mạng? Bởi vì khi một kẻ tấn công tìm cách đột nhập vào hệ thống, chúng ít khi kiên nhẫn ngồi nghe lén từng gói tin HTTPS. Thay vào đó, mục tiêu tối thượng của chúng là đánh cắp được file snapshot đĩa cứng, trộm được file backup database hoặc dump toàn bộ bảng dữ liệu người dùng. Nếu dữ liệu tĩnh ở dạng văn bản thô (Plaintext), hậu quả sẽ là thảm họa lộ lọt thông tin quy mô hàng triệu người dùng: số thẻ tín dụng, số định danh cá nhân (CCCD/SSN), hồ sơ bệnh án và lịch sử giao dịch.

Tuy nhiên, một sai lầm phổ biến là nhiều kỹ sư chỉ bật tính năng mã hóa ổ đĩa (Full Disk Encryption) rồi cho rằng hệ thống đã tuyệt đối an toàn. Thực tế, khi máy chủ đang chạy (system mounted), bất kỳ ai có quyền root OS hoặc tài khoản DBA đều có thể đọc toàn bộ plaintext!

Để giải quyết triệt để bài toán này, chúng ta bắt buộc phải áp dụng nguyên tắc **Phòng thủ đa tầng (Defense in Depth)**: thiết lập ba lớp rào chắn độc lập từ cấp phân vùng ổ đĩa (Block/Volume Level), cấp phong bì tập tin sao lưu (File/Envelope Level), cho đến cấp từng cột dữ liệu nhạy cảm bên trong cơ sở dữ liệu (Database Column Level). Trong bài viết này, mình sẽ cùng các bạn tìm hiểu chi tiết và thực hành từng lớp phòng thủ bằng VeraCrypt, OpenSSL và extension `pgcrypto` trong PostgreSQL.

---

# II. Kiến trúc & So sánh thực tế

### 1. Mô hình Phòng thủ 3 Tầng cho Dữ liệu Data-at-Rest

Mô hình phòng thủ toàn diện kết hợp ba tầng mã hóa độc lập nhằm vô hiệu hóa các vector tấn công ở từng cấp độ:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ MÔ HÌNH PHÒNG THỦ ĐA TẦNG CHO DỮ LIỆU DATA-AT-REST                                     │
├────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                        │
│ [TẦNG 1: MÃ HÓA KHỐI / PHÂN VÙNG (Block / Volume Level)]                               │
│  Công nghệ: VeraCrypt / Linux LUKS (AES-256-XTS)                                       │
│  Mục tiêu: Mã hóa toàn bộ partition ảo /mnt/secure_vault                               │
│  Đối phó: Trộm cắp ổ cứng vật lý, sao chép trái phép raw image / EBS snapshot          │
│                                                                                        │
│        ▲                                                                               │
│        │ (Dữ liệu bên trong Volume)                                                    │
│                                                                                        │
│ [TẦNG 2: MÃ HÓA BAO THƯ TẬP TIN (File / Envelope Level)]                               │
│  Công nghệ: OpenSSL CLI Envelope (RSA-4096 Key Wrapping + AES-256-GCM DEK)             │
│  Mục tiêu: Đóng gói file backup dữ liệu thành phong bì mã hóa độc lập                  │
│  Đối phó: Quản trị viên OS không có Private Key, rò rỉ file backup trên S3             │
│                                                                                        │
│        ▲                                                                               │
│        │ (Dữ liệu trong Database)                                                      │
│                                                                                        │
│ [TẦNG 3: MÃ HÓA CỘT CƠ SỞ DỮ LIỆU (Database Column Level)]                             │
│  Công nghệ: PostgreSQL extension pgcrypto (pgp_sym_encrypt / pgp_pub_encrypt)          │
│  Mục tiêu: Mã hóa từng ô dữ liệu nhạy cảm (Số CCCD, Số thẻ tín dụng, Lương)            │
│  Đối phó: SQL Injection, DBA tò mò xem trộm, File pg_dump bị thất thoát                │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

1. **Tầng 1 (Block/Volume Encryption - VeraCrypt / LUKS)**:
   - Sử dụng chế độ thuật toán `AES-256-XTS` chuẩn hóa cho lưu trữ khối.
   - Bảo vệ hệ thống trước nguy cơ mất trộm vật lý ổ đĩa, đánh cắp phân vùng ảo hoặc nhân bản snapshot máy chủ. Khi máy chủ tắt nguồn hoặc volume unmount, toàn bộ dữ liệu chỉ là một khối bit ngẫu nhiên.
2. **Tầng 2 (File/Envelope Encryption - OpenSSL)**:
   - Áp dụng nguyên lý Mã hóa Phong bì (Envelope Encryption) kết hợp giữa mã hóa đối xứng (Symmetric) và bất đối xứng (Asymmetric).
   - Dữ liệu lớn được mã hóa cực nhanh bằng khóa phiên ngẫu nhiên `DEK` (Data Encryption Key) theo thuật toán `AES-256-GCM`. Sau đó, chính chiếc khóa DEK này được bọc lại bằng khóa công khai `Master Public Key` (RSA-4096). Nhờ đó, file backup khi xuất ra và lưu trữ trên Cloud Storage (S3/GCS) được bảo vệ hoàn toàn: chỉ những ai nắm giữ `Master Private Key` trong két sắt an toàn mới giải mã được.
3. **Tầng 3 (Column-level Database Encryption - PostgreSQL `pgcrypto`)**:
   - Mã hóa trực tiếp từng trường dữ liệu nhạy cảm (như số CMND/CCCD, thẻ ngân hàng) bằng các hàm mã hóa PGP trong PostgreSQL.
   - Ngay cả khi hacker khai thác được lỗ hổng SQL Injection để đọc bảng hoặc kẻ trộm có được bản backup dạng `pg_dump`, chúng chỉ nhận được các chuỗi nhị phân mã hóa mà không thể biết được nội dung thực tế nếu không có khóa bí mật ứng dụng.

### 2. Bảng so sánh 3 tầng mã hóa

| Tiêu chí | Tầng 1: VeraCrypt Volume | Tầng 2: OpenSSL Envelope | Tầng 3: PostgreSQL pgcrypto |
| :--- | :--- | :--- | :--- |
| **Phạm vi bảo vệ** | Toàn bộ phân vùng ổ đĩa ảo | Từng file riêng lẻ (Backup/Export) | Từng ô (cell/column) trong bảng DB |
| **Thuật toán cốt lõi** | AES-256-XTS | RSA-4096 + AES-256-GCM | PGP Symmetric / Asymmetric (AES/Blowfish)|
| **Vị trí giải mã** | Kernel Device Mapper / Driver | RAM ứng dụng / Script giải mã | Bên trong PostgreSQL Engine |
| **Bảo vệ trước ai?** | Kẻ trộm ổ cứng vật lý | Người truy cập được file backup S3 | DBA, SQL Injection, rò rỉ pg_dump |
| **Tác động hiệu năng** | Rất thấp (hỗ trợ bởi CPU AES-NI) | Trung bình (chỉ khi backup/export) | Cao trên câu lệnh SQL truy vấn |
| **Đánh B-tree Index** | Hoàn toàn bình thường | Không áp dụng | Bị vô hiệu hóa (cần Blind Index) |

---

# III. Cài đặt & Thực hành code: Triển khai thực chiến

### 1. Triển khai Tầng 1: Khởi tạo và Mount Phân vùng mã hóa VeraCrypt Headless trên Linux

Script dưới đây tự động hóa quá trình tạo một container mã hóa 2GB và mount vào thư mục hệ thống mà không cần giao diện đồ họa:

```bash
#!/usr/bin/env bash
# secure_volume_setup.sh: Tạo container mã hóa VeraCrypt không cần GUI
set -euo pipefail

VAULT_CONTAINER="/var/secure_storage/vault.hc"
MOUNT_POINT="/mnt/secure_vault"
VOLUME_SIZE="2G"

# Tạo thư mục lưu trữ
mkdir -p /var/secure_storage "$MOUNT_POINT"

# 1. Tạo file container mã hóa 2GB bằng thuật toán AES, hàm băm SHA-512
echo "Khởi tạo VeraCrypt Volume..."
veracrypt --text --create "$VAULT_CONTAINER" \
  --size="$VOLUME_SIZE" \
  --password="MySuperSecurePassphrase123!@#" \
  --encryption=AES \
  --hash=sha512 \
  --filesystem=ext4 \
  --volume-type=normal \
  --random-source=/dev/urandom

# 2. Mount phân vùng vào thư mục làm việc
echo "Mounting volume vào $MOUNT_POINT..."
veracrypt --text --mount "$VAULT_CONTAINER" "$MOUNT_POINT" \
  --password="MySuperSecurePassphrase123!@#" \
  --protect-hidden=no

# Kiểm tra trạng thái
df -h "$MOUNT_POINT"
echo "Phân vùng an toàn đã sẵn sàng lưu trữ dữ liệu nhạy cảm!"
```

### 2. Triển khai Tầng 2: Tự động hóa OpenSSL Envelope Encryption

Script dưới đây thực thi chu trình mã hóa phong bì số (Digital Envelope): sinh ngẫu nhiên DEK, mã hóa dữ liệu với AES-256-GCM, bọc DEK bằng RSA-4096 và đóng gói vào file tar duy nhất:

```bash
#!/usr/bin/env bash
# envelope_encrypt.sh: Mã hóa file theo mô hình Digital Envelope
set -euo pipefail

INPUT_FILE="customer_financial_records.csv"
OUTPUT_PACKAGE="customer_records.env.tar"
MASTER_PUB_KEY="master_public.pem"

# 1. Sinh khóa bất đối xứng Master Key RSA-4096 (nếu chưa có)
if [ ! -f "$MASTER_PUB_KEY" ]; then
  openssl genpkey -algorithm RSA -out master_private.pem -pkeyopt rsa_keygen_bits:4096
  openssl rsa -pubout -in master_private.pem -out "$MASTER_PUB_KEY"
fi

# 2. Sinh ngẫu nhiên Data Encryption Key (DEK) 256-bit từ CSPRNG hệ điều hành
openssl rand -out /tmp/dek.bin 32
openssl rand -out /tmp/iv.bin 12

# 3. Mã hóa dữ liệu bằng AES-256-GCM với DEK
openssl enc -aes-256-gcm -in "$INPUT_FILE" -out /tmp/data.enc \
  -K $(xxd -p -c 32 /tmp/dek.bin) \
  -iv $(xxd -p -c 12 /tmp/iv.bin)

# 4. Dùng Master Public Key để mã hóa bọc (wrap) chiếc khóa DEK
openssl pkeyutl -encrypt -pubin -inkey "$MASTER_PUB_KEY" \
  -in /tmp/dek.bin -out /tmp/dek.enc

# 5. Đóng gói Ciphertext, Wrapped DEK và IV vào một gói duy nhất
tar -cvf "$OUTPUT_PACKAGE" -C /tmp data.enc dek.enc iv.bin

# Dọn dẹp khóa DEK tạm thời khỏi bộ nhớ/đĩa cứng
shred -u /tmp/dek.bin /tmp/iv.bin /tmp/data.enc /tmp/dek.enc

echo "File đã được mã hóa Envelope an toàn thành công: $OUTPUT_PACKAGE"
```

### 3. Triển khai Tầng 3: Mã hóa Cột cơ sở dữ liệu với PostgreSQL `pgcrypto` và Kỹ thuật Blind Index

Khi mã hóa một cột trong database, việc tìm kiếm chính xác (`WHERE ssn = '...'`) trở nên bất khả thi vì mỗi lần mã hóa ra một ciphertext khác nhau. Dưới đây là giải pháp kết hợp **pgcrypto** với kỹ thuật **Blind Index** (dùng HMAC-SHA256) giúp tìm kiếm O(1) qua B-Tree Index:

```sql
-- Kích hoạt extension mật mã học
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Tạo bảng lưu trữ thông tin khách hàng nhạy cảm
DROP TABLE IF EXISTS customer_secure_vault;
CREATE TABLE customer_secure_vault (
    id SERIAL PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    -- Cột lưu trữ dưới dạng bytea mã hóa
    ssn_encrypted BYTEA NOT NULL,
    credit_card_encrypted BYTEA NOT NULL,
    -- Cột Blind Index phục vụ tìm kiếm chính xác mà không cần giải mã
    ssn_bindex VARCHAR(64) NOT NULL
);

-- Tạo Index trên cột Blind Index để tìm kiếm O(1) qua B-tree
CREATE INDEX idx_customer_ssn_bindex ON customer_secure_vault (ssn_bindex);

-- Thiết lập khóa bí mật trong phiên làm việc của ứng dụng (Application Session Secret)
-- Trong thực tế được inject qua SET LOCAL từ connection pool sau khi xác thực
SET LOCAL app.secret_key = 'K1#9mZ$8vL2!pQx@7wR4_EnterpriseSecretPassphrase';
SET LOCAL app.hmac_salt  = 'FixedSaltForBlindIndexing_99482';

-- 1. Thao tác INSERT: Mã hóa cột nhạy cảm và sinh Blind Index
INSERT INTO customer_secure_vault (
    full_name, 
    ssn_encrypted, 
    credit_card_encrypted, 
    ssn_bindex
) VALUES (
    'Trần Phú Vinh',
    -- Mã hóa AES-256 qua PGP với IV ngẫu nhiên cho mỗi row
    pgp_sym_encrypt('079194001234', current_setting('app.secret_key'), 'cipher-algo=aes256, compress-algo=0'),
    pgp_sym_encrypt('4111-2222-3333-4444', current_setting('app.secret_key'), 'cipher-algo=aes256'),
    -- Sinh HMAC-SHA256 làm Blind Index tìm kiếm (Deterministic Token)
    encode(hmac('079194001234', current_setting('app.hmac_salt'), 'sha256'), 'hex')
);

-- 2. Thao tác SELECT giải mã dữ liệu hiển thị cho người có thẩm quyền
SELECT 
    id,
    full_name,
    pgp_sym_decrypt(ssn_encrypted, current_setting('app.secret_key')) AS ssn_decrypted,
    pgp_sym_decrypt(credit_card_encrypted, current_setting('app.secret_key')) AS credit_card_decrypted
FROM customer_secure_vault;

-- 3. Thao tác TÌM KIẾM BẰNG BLIND INDEX (Tốc độ cực nhanh nhờ Index B-Tree mà không lộ Plaintext)
SELECT 
    id,
    full_name,
    pgp_sym_decrypt(ssn_encrypted, current_setting('app.secret_key')) AS ssn
FROM customer_secure_vault
WHERE ssn_bindex = encode(hmac('079194001234', current_setting('app.hmac_salt'), 'sha256'), 'hex');
```

---

# IV. Lesson learned & Tổng kết: Best Practices & Cạm bẫy production

Từ thực tế vận hành các hệ thống ngân hàng và thanh toán trực tuyến, mình xin chia sẻ 4 cạm bẫy mật mã học mà các bạn bắt buộc phải nằm lòng:

1. **Tuyệt đối tránh xa chế độ AES-ECB (Electronic Codebook)**:
   - Chế độ ECB mã hóa từng khối dữ liệu 16-byte độc lập với cùng một khóa bí mật. Điều này khiến các mẫu dữ liệu lặp lại trong plaintext sẽ sinh ra các khối ciphertext y hệt nhau (bức tranh nổi tiếng về chú chim cánh cụt Tux vẫn hiện rõ đường nét sau khi mã hóa ECB). Hãy luôn dùng chế độ xác thực toàn vẹn AEAD như `AES-GCM` hoặc `AES-XTS`.
2. **Kỹ thuật Blind Index là chìa khóa để tìm kiếm**:
   - Đừng bao giờ lưu khóa mã hóa deterministic yếu chỉ để tìm kiếm được trong database. Kỹ thuật Blind Index với HMAC-SHA256 và một Secret Salt tách biệt là tiêu chuẩn vàng của ngành để vừa mã hóa mạnh (Probabilistic Encryption) vừa tìm kiếm nhanh qua B-Tree Index.
3. **Quản trị vòng đời khóa mật mã (Key Rotation)**:
   - Đừng hardcode passphrase vào file script hay mã nguồn Git. Hãy ủy thác việc quản lý Master Key cho các dịch vụ chuyên dụng như HashiCorp Vault, AWS KMS hay Azure Key Vault. Khi cần xoay khóa (Key Rotation), mô hình Envelope Encryption chỉ đòi hỏi giải mã và bọc lại khóa DEK mà không cần phải mã hóa lại hàng Terabyte dữ liệu tĩnh!
4. **Hiểm họa rò rỉ Plaintext qua Swap Partition và Memory Dump**:
   - Dữ liệu khi giải mã vào RAM có thể bị hệ điều hành hoán chuyển (swap) xuống đĩa cứng dưới dạng văn bản thô. Hãy luôn cấu hình mã hóa phân vùng swap (`cryptswap`) hoặc sử dụng hàm `mlock()` trong ứng dụng để ghim chặt các trang bộ nhớ chứa thông tin nhạy cảm.

Bằng cách áp dụng triệt để chiến lược phòng thủ 3 tầng: VeraCrypt (khối) $\rightarrow$ OpenSSL Envelope (tập tin) $\rightarrow$ PostgreSQL pgcrypto (cột dữ liệu), các bạn đã tạo nên một pháo đài mật mã học kiên cố, bảo vệ an toàn tối đa cho tài sản dữ liệu của doanh nghiệp!
