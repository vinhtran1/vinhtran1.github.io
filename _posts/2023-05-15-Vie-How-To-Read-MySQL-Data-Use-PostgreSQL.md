---
title: 'Dùng PostgreSQL để đọc dữ liệu MySQL'
date: 2023-05-15 11:55:00 +0700
categories: [Tutorial]
tags: [PostgreSQL, MySQL]
pin: false
---
## Mở đầu

Ở bài trước ([Link](https://vinhtran1.github.io/posts/Vie-Foreign-Data-Wrapper-Tutorial/)) có giới thiệu cách để một database PostgreSQL có thể kết nối và lấy data của một database PostgreSQL khác. Trong bài này sẽ tiếp tục giới thiệu cách mà PostgreSQL kết nối và lấy dữ liệu của một database MySQL cũng bằng cách tạo ra các foreign table. Extension đó có tên là `mysql_fdw`

Lưu ý là extension này chưa được PostgreSQL Global Development Group (PGDG - là tổ chức đã tạo ra PostgreSQL) hỗ trợ chính thức, nên cần cẩn thận khi sử dụng

## Bước 0: Setup môi các tool cần thiết

### Bước 0.1: Setup database MySQL

```bash
podman run --name db_mysql -e MYSQL_ROOT_PASSWORD=password_value -p 3306:3306 -d mysql
```

Thông tin truy cập của MySQL:

- Host: 127.0.0.1
- Port: 3306
- Username: root
- Password: password_value
- Database: mysql

Tạo trước một bảng trong database

```sql
create database sampledb;
CREATE TABLE sampledb.sample_table (
  text_column varchar(100) DEFAULT NULL,
  number_column int DEFAULT NULL
);
```

Insert dữ liệu mẫu

```sql
INSERT INTO sampledb.sample_table (text_column, number_column) VALUES('value1', 1);
INSERT INTO sampledb.sample_table (text_column, number_column) VALUES('value2', 2);
INSERT INTO sampledb.sample_table (text_column, number_column) VALUES('value3', 3);
```

### Bước 0.2: Setup database PostgreSQL đã được cài đặt sẵn mysql_fdw

Ở bài viết này chỉ tập trung vào việc setup foreign table nên sẽ không đi vào chi tiết cách cài đặt mysql_fdw lên server PostgreSQL. Sẽ có một bài viết khác hướng dẫn cách cài đặt sau.

Đầu tiên clone repo github

```bash
git clone https://github.com/chumaky/docker-images.git
cd docker-images
```

Cách setup đã có hướng dẫn trong repo. Ở đây chỉ ghi lại

Build Image và Run

```bash
podman build -t postgres_mysql -f postgres_mysql.docker
podman run -d --name pg_fdw_test -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres_mysql
```

Thông tin truy cập của PostgreSQL:

- Host: 127.0.0.1
- Port: 5432
- Username: postgres
- Password: postgres
- Database: postgres

Sau đó vào database bằng cách:

```bash
podman exec -it pg_fdw_test psql postgres postgres
```

## Bước 1: Tạo extension

Các bước sau đây đều thực hiện trên database PostgreSQL

Lưu ý chỉ các user có role superuser mới thực hiện được 

```sql
CREATE EXTENSION mysql_fdw;
```

## Bước 2: Tạo server

```sql
CREATE SERVER mysql_server
	FOREIGN DATA WRAPPER mysql_fdw
	OPTIONS (host '127.0.0.1', port '3306');
```

## Bước 3: Tạo user mapping

```sql
CREATE USER MAPPING FOR current_user
SERVER mysql_server
OPTIONS (username 'root', password 'password_value');
```

Chỉ user nào được tạo user mapping thì mới có thể lấy được data của foreign table. Nên cần thực hiện câu này với mỗi user cần lấy data

## Bước 4: Tạo foreign table

```sql
CREATE FOREIGN TABLE foreign_sample_table -- có thể đặt lại thành tên bảng khác
( 
  text_column varchar(100) DEFAULT NULL,
  number_column int4 DEFAULT NULL
)
SERVER mysql_server
	OPTIONS (dbname 'sampledb', table_name 'sample_table');
```

Vậy là xong, chạy thử thôi

```sql
select * from foreign_sample_table

text_column|number_column|
-----------+-------------+
value1     |            1|
value2     |            2|
value3     |            3|
```

## Kết luận

Tương tự với việc dùng `postgres_fdw`, các ưu điểm, khuyết điểm của việc dùng `mysql_fdw` này là:

### Ưu điểm

- Có thể được sử dụng như là một phương pháp stream data từ MySQL đến PostgreSQL theo thời gian thực
- Không chiếm dung lượng ổ đĩa của PostgreSQL

### Khuyết điểm

- Không được hỗ trợ chính thức bởi PGDG, nên có khả năng xảy ra lỗi không mong muốn mà không được hỗ trợ nhanh chóng
- Cách setup có phần phức tạp nếu không dùng Podman, Docker
- Tốc độ truy vấn của foreign table sẽ chậm hơn physical table, thể hiện rất rõ khi cần JOIN nhiều bảng có hàng trăm nghìn dòng dữ liệu trở lên

Như vậy bài viết này đã trình bày việc dùng PostgreSQL để kết nối và đọc dữ liệu của MySQL. Hy vọng bài này có ích đến mọi người