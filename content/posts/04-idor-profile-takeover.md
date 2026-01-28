---
title: "Chiếm quyền tài khoản bằng IDOR trên API hồ sơ"
date: 2026-01-28T20:30:00+07:00
draft: false
description: "Một lỗi tưởng chừng nhỏ trong API update profile dẫn đến account takeover."
featuredImage: "/img/blowfish_banner.png"

categories: ["Pentest", "Web Application"]
tags: ["idor", "access-control", "api", "authorization"]

showTableOfContents: true
showAuthor: true
showDate: true
showReadingTime: true
---

## 1. Giới thiệu

IDOR (Insecure Direct Object Reference) thuộc nhóm **Broken Access Control** trong OWASP Top 10.  
Trong case này, chỉ một field `user_id` trong body request đã đủ để chiếm quyền tài khoản người khác.

## 2. API update profile

Request bình thường:

```http
POST /api/v1/profile/update HTTP/1.1
Content-Type: application/json
Authorization: Bearer <JWT_USER_123>

{
  "user_id": 123,
  "full_name": "User One",
  "phone": "0900000001"
}
```

Server chỉ kiểm tra:

- Token hợp lệ.
- `user_id` có tồn tại trong DB.

Không kiểm tra `user_id` trong body có trùng với user trong token hay không.

## 3. Khai thác IDOR

Attacker đổi `user_id`:

```json
{
  "user_id": 5,
  "full_name": "Pwned Account",
  "phone": "0900000005"
}
```

Nếu request trả về `200 OK` và thông tin user 5 thay đổi → **IDOR confirmed**.

Tùy endpoint, attacker có thể:

- Đổi email, số điện thoại.
- Enable 2FA/disable 2FA.
- Gắn tài khoản mạng xã hội của attacker.

## 4. Bài học cho design API

- **Không tin bất kỳ identifier nào client gửi lên** (user_id, account_id, order_id…).
- Luôn derive identifier từ:
  - Token (JWT claim).
  - Session server-side.
- Với mọi action nhạy cảm, cần **re-check ownership** resource trước khi update.

## 5. Kết luận

IDOR là lỗi “nhẹ nhàng” nhưng impact rất nặng vì:

- Dễ bị bỏ qua khi review.
- Không tạo lỗi rõ ràng, chỉ âm thầm sửa data.

Khi pentest API, hãy luôn thử:

- Đổi ID sang của user khác.
- Đổi sang ID không tồn tại.
- So sánh response và hành vi hệ thống trước/sau.

