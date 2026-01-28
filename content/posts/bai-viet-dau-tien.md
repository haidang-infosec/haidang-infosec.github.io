---
title: "Phân tích lỗ hổng trang SNL The Exhibition"
date: 2026-01-28T15:00:00+07:00
draft: false
description: "Phát hiện lỗi PHP Object Injection trên WordPress 5.4.10"

# PHÂN LOẠI (Quan trọng)
categories: ["Pentest", "Real-world"]  # Chuyên mục lớn
tags: ["wordpress", "cve", "wpscan", "wordops"] # Từ khóa nhỏ

# Cấu hình giao diện bài viết (Của Blowfish)
showTableOfContents: true # Hiện mục lục
showAuthor: true
showDate: true
showReadingTime: true
---

## 1. Giới thiệu
Hôm nay tôi đã thực hiện pentest trang web này...

## 2. Reconnaissance
Sử dụng công cụ `wpscan`, tôi phát hiện:

{{< alert icon="fire" >}}
**Cảnh báo:** Trang web sử dụng WordPress 5.4.10 đã lỗi thời 4 năm!
{{< /alert >}}

### Code dính lỗi
Dưới đây là đoạn code PHP gây lỗi:

```php
// Ví dụ về lỗ hổng
$data = unserialize($_GET['data']);