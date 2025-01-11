---
title: 'System Design: Thiết kế một hệ thống lưu trữ password'
date: 2023-05-17 17:25:00 +0700
categories: [Blogging]
tags: [System Design]
pin: false
---


## Bài toán

Trong cơ sở dữ liệu (CSDL), dữ liệu cá nhân nói chung và mật khẩu đăng nhập nói riêng là các dữ liệu cần đặc biệt quan tâm khi lưu trữ, xử lý. Bài toán đặt ra là nên thiết kế CSDL như thế nào để lưu trữ chúng an toàn, bảo mật

Trong bài viết này sẽ trình bày một số cách lưu trữ mật khẩu với mức độ bảo mật tăng dần. Biết được những cách này có thể sử dụng để tùy chọn cách để lưu trữ các dữ liệu cá nhân khác.

## Level 1: PLain Text

Ở mức độ này, mật khẩu được lưu trực tiếp vào database. Đây là cách lưu trữ đơn giản nhất, dễ thực hiện nhất. Dễ thấy đây là cách lưu trữ không có gì bảo mật cả, ngoại trừ một lớp mật khẩu để truy cập vào CSDL

Khi lưu mật khẩu dưới dạng plain text, việc xác thực sẽ diễn ra bằng câu query

```sql
select data
from personal_data_table
where username = username_value -- thay username_value là giá trị được nhập
and password = password_value -- thay password_value là giá trị được nhập vào
```

Và điều này sẽ dẫn đến một lỗ hổng bảo mật dễ bị khai thác là SQL Injection. Vì vậy, cách lưu trữ này không thích hợp cho bất kỳ trường hợp lưu trữ dữ liệu cá nhân, dữ liệu nhạy cảm nào.

## Level 2: Encrypting

Ở mức độ bảo mật này, các thông tin quan trọng sẽ được biến đổi thành một chuỗi không có ý nghĩa bằng một chuỗi gọi là key và một thuật toán mã hóa. Chuỗi này hoàn toàn có thể được biến đổi trở lại thành chuỗi ban đầu nếu biết được.


![encryption.png](https://images2.imgbox.com/8e/d6/ZgXQs5u2_o.png)
Như vậy mức độ bảo mật đã được tăng lên. Hacker để lấy được mật khẩu cần phải biết được

- Key mã hóa/giải mã
- Thuật toán mã hóa
- Chuỗi đã được mã hóa

Tuy nhiên, nếu có nhiều key mã hóa, thì các key này vẫn sẽ được lưu vào một nơi nào đó để có thể lấy ra và mã hóa, nên vẫn có rủi ro xuất hiện khi CSDL chứa key bị hacker tấn công và lấy mất

## Level 3: Hashing

Cách này làm giảm thiểu rủi ro việc mất key bằng cách… không lưu key. Một hàm băm (hash) được dùng để hash chuỗi giá trị ban đầu ra thành một chuỗi mới. Và chuỗi mới này, về mặt kỹ thuật lập trình thì, không thể đảo ngược lại thành chuỗi ban đầu cho dù có biết được thuật toán hash đi chăng nữa
![hash.png](https://images2.imgbox.com/e6/c4/9sMUDDZw_o.png)

Có rất nhiều hàm hash phổ biến hiện nay như MD5, SHA-1, SHA-256,… Các bạn có thể tham khảo thêm tại link [Wikipedia](https://en.wikipedia.org/wiki/List_of_hash_functions)

Lúc này, các hệ thống xác thực sẽ không so sánh trực tiếp giá trị password mà là so sánh chuỗi sau khi được hash. Bằng cách này, rủi ro về kỹ thuật SQL Injection cũng biến mất

Tuy nhiên, với tính chất căn bản của hàm hash là một chuỗi input sẽ cho ra một chuỗi output duy nhất (hoặc xác suất trùng nhau cực nhỏ), hoàn toàn có thể “giải mã” được chuỗi output nếu chuỗi đó đã được tính toán từ trước và lưu lại cặp giá trị input ouptut

Ví dụ:
Với hàm hash là MD5, chuỗi hash của chuỗi “**hello**" sẽ là chuỗi “**5d41402abc4b2a76b9719d911017c592”**

Copy chuỗi này và paste vào ô bên trong trang web [https://crackstation.net/](https://crackstation.net/)

Kết quả sẽ là chuỗi ‘**hello**’ như hình

![crackstation.png](https://images2.imgbox.com/8a/6d/vioUZ78N_o.png)

Như vậy, việc đặt mật khẩu, hoặc các chuỗi quá đơn giản, cộng thêm hàm hash quá phổ biến thì cách này cũng hoàn toàn có rủi ro bị lấy mất dữ liệu

CSDL lưu các bộ giá trị input, type, result được gọi là rainbow table

Hàm hash không thích hợp khi hệ thống có nhu cầu đọc các giá trị (ví dụ họ tên, địa chỉ cần phải được đọc hiểu để có thể giao hàng chính xác) mà chỉ nên dùng kỹ thuật encrypt. Hàm hash thường dùng cho hệ thống khi lưu password nhiều hơn 

## Level 4: Salting

Vấn đề của level 3 là khi các thông tin cần bảo mật lại là các chuỗi quá đơn giản sẽ dễ dàng bị suy đoán ra. Vậy để khắc phục vấn đề này, ta có thể thêm vào một chuỗi ký tự đủ dài, đủ khó đoán mà bảng brute force chưa hề gặp qua. Kỹ thuật này gọi là salting.

Ta thêm vào chuỗi gốc một chuỗi ngẫu nhiên bất kỳ, có đủ các ký tự in hoa, thường, chữ số, ký tự đặc biệt và có độ dài đủ lớn. Chuỗi này được gọi là salt. Như vậy sẽ đảm bảo chuỗi giá trị này sẽ không phải là một chuỗi phổ biến được tính toán sẵn trong rainbow table

Với việc lặp lại level 4 và 3 nhiều lần, ví dụ ta có thể thêm salt vào phía trước chuỗi gốc, rồi sau đó dùng hàm hash, rồi lại thêm một salt khác vào sau chuỗi gốc, rồi lại dùng hàm hash khác để hash. Thực hiện càng nhiều lần như vậy dữ liệu sẽ càng được bảo mật cao, cho dù có sử dụng rainbow table đi chăng nữa. Nếu muốn lấy dữ liệu, hacker phải lập lại một rainbow table mới, một công việc mất rất nhiều thời gian

Lưu ý, các chuỗi salt nên được khởi tạo dành riêng cho mỗi chuỗi giá trị. Thông thường salt được lưu cùng với chuỗi đầu ra mà nó đã tham gia mã hóa

Muốn bảo mật hơn nữa không, hãy đến với level kế tiếp

## Level 5: Pepper

Ở level trước đã thêm muối, giờ thêm tiêu nhé

Giống như salt, pepper là một chuỗi ngẫu nhiên được thêm vào một chuỗi trước khi chuỗi đó được hash/encrypt. Nhưng khác với salt, pepper là một chuỗi được áp dụng cho toàn bộ hệ thống thay vì tạo mới liên tục cho mỗi dữ liệu cần được hash/encrypt. Điểm khác biệt kết tiếp là pepper thường được lưu ở một hệ thống khác, được bảo vệ bằng nhiều lớp xác thực mật khẩu, VPN, phân quyền,… đảm bảo chỉ có tài khoản có liên quan mới lấy được pepper. 

## Kết luận

Như vậy, bằng cách kết hợp các kỹ thuật đã trình bày ở trên, cộng thêm một số kỹ thuật bảo mật như VPN, giới hạn địa chỉ IP truy cập, xác thực 2 lớp (2FA),… CSDL đã bảo mật hơn rất nhiều