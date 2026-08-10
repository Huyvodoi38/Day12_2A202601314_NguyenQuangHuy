# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục           | Nội dung                                                      |
| -------------- | -------------------------------------------------------------- |
| Họ và tên   | Nguyễn Quang Huy                                              |
| Mã học viên | 2A202601314                                                    |
| Repo           | https://github.com/Huyvodoi38/Day12_2A202601314_NguyenQuangHuy |

## Service

| Mục         | Nội dung                            |
| ------------ | ------------------------------------ |
| Public URL   | https://day12-chat-ecks.onrender.com |
| Platform     | Render                               |
| Ngày deploy | 2026-08-10                           |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến                 | Đã set | Ghi chú                                      |
| --------------------- | -------- | --------------------------------------------- |
| `PORT`              | ✅       | platform tự gán                             |
| `API_TOKEN`         | ✅       | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL`         | ✅       | Redis Instance trên Render                   |
| `BUCKET_CAPACITY`   | ✅       | 10                                            |
| `REFILL_PER_MINUTE` | ✅       | 10                                            |
| `DAILY_BUDGET_USD`  | ✅       | 1.0                                           |
| `LOG_LEVEL`         | ✅       | INFO                                          |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-chat-ecks.onrender.com/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-chat-ecks.onrender.com/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST https://day12-chat-ecks.onrender.com/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-chat-ecks.onrender.com/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-chat-ecks.onrender.com/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
curl.exe -i https://day12-chat-ecks.onrender.com/healthz
HTTP/1.1 200 OK
Date: Mon, 10 Aug 2026 07:56:19 GMT
Content-Type: application/json
status: ok

curl.exe -i https://day12-chat-ecks.onrender.com/readyz
HTTP/1.1 200 OK
Date: Mon, 10 Aug 2026 07:56:40 GMT
Content-Type: application/json
status: ready, redis: true

huyqu@LTA MINGW64 ~/Desktop/ai-thuc-chien/Day12_2A202601314_NguyenQuangHuy (main)
$ curl -i -X POST https://day12-chat-ecks.onrender.com/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'
HTTP/1.1 401 Unauthorized
Date: Mon, 10 Aug 2026 08:04:15 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 423ac5e9-ba1b-4ad9
Server: cloudflare
vary: Accept-Encoding
www-authenticate: Bearer
x-render-origin-server: uvicorn
CF-RAY: a28d7d1c8ca60721-HKG
alt-svc: h3=":443"; ma=86400

{"detail":"invalid or missing bearer token"}

huyqu@LTA MINGW64 ~/Desktop/ai-thuc-chien/Day12_2A202601314_NguyenQuangHuy (main)
$ curl -i -X POST https://day12-chat-ecks.onrender.com/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 1MCse0U5YmZH-DQKWOVw-bSjCQZp_zY2ywHo_HobtVk" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy la gi?"}'
HTTP/1.1 200 OK
Date: Mon, 10 Aug 2026 08:16:34 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 9d56bbdd-4a56-4694
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a28d8f27dad5dc87-HKG
alt-svc: h3=":443"; ma=86400

{"reply":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên.","client_id":"sv-test","turns_before":0,"usd_cost":2.265e-05,"usage":{"prompt":3,"completion":37}}

huyqu@LTA MINGW64 ~/Desktop/ai-thuc-chien/Day12_2A202601314_NguyenQuangHuy (main)
$ for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-chat-ecks.onrender.com/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer 1MCse0U5YmZH-DQKWOVw-bSjCQZp_zY2ywHo_HobtVk" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
200 200 200 200 200 200 200 200 200 200 429 200 429 429 429 
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl
