---
title: "Blind SQLi âm thầm trong form login"
date: 2026-01-28T20:20:00+07:00
draft: false
description: "Từ khác biệt thời gian phản hồi đến khai thác blind SQLi trên chức năng đăng nhập."
featuredImage: "/img/fireflies.svg"

categories: ["Pentest", "Web Application"]
tags: ["sqli", "blind-sqli", "login", "burp"]

showTableOfContents: true
showAuthor: true
showDate: true
showReadingTime: true
---

## 1. Triệu chứng ban đầu

Trong quá trình test form login, mình nhận thấy:

- Khi nhập user hợp lệ + sai mật khẩu → response ~300ms.
- Khi nhập user không tồn tại → response ~50ms.

Sự khác biệt tuy nhỏ nhưng **ổn định**, đủ để nghi ngờ backend đang thực hiện query nặng hơn cho user tồn tại.

## 2. Thử tiêm payload đơn giản

Request login:

```http
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=admin&password=test
```

Thử payload:

```text
username=admin' AND '1'='1&password=test
```

Ứng dụng trả về lỗi 500 → có khả năng dính SQLi.  
Sau khi tinh chỉnh theo kiểu **time-based blind**:

```text
username=admin' AND SLEEP(3)-- -&password=test
```

Response delay ~3s, chứng minh có SQLi.

## 3. Khai thác có hệ thống bằng Burp

Mục tiêu:

- Trích xuất **length** và **nội dung** password hash của user `admin`.

Ý tưởng:

- Dùng `SLEEP()` dựa trên từng điều kiện true/false.

Ví dụ kiểm tra ký tự thứ 1 có phải là `a`:

```sql
admin' AND IF(SUBSTR(password,1,1)='a',SLEEP(3),0)-- -
```

Trong Burp Intruder:

- Position: ký tự cần brute-force.
- Payload: charset `[0-9a-f]` (nếu hash hex).
- Grep: thời gian phản hồi > 3s → điều kiện đúng.

## 4. Phòng thủ

- Tuyệt đối dùng **prepared statements / parameterized queries**.
- Không bao giờ nối chuỗi user input vào query.
- Với form login, nên có:
  - **Generic error message** (“Sai thông tin đăng nhập”) cho mọi case.
  - **Rate limiting** để giảm bề mặt brute-force.

## 5. Kết luận

Blind SQLi thường không “đập vào mắt” như classic SQLi, nhưng chỉ cần:

- Quan sát **time response** cẩn thận.
- Sử dụng đúng tool (Burp Intruder, SQLMap với cấu hình hợp lý).

…là có thể biến một form login bình thường thành **gateway vào database**.

