# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Doãn Hữu Nguyên |
| Mã học viên | 2A202602671 |
| Repo | https://github.com/nace1504/K4-L3B-DAY12-DoanHuuNguyen-2A202602671-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-9efc.up.railway.app |
| Platform | Railway (project `day12-agent`: service `agent` build từ Dockerfile + service `Redis`) |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt bằng `railway variable set`, khóa riêng cho cloud (khác khóa local), không nằm trong repo |
| `REDIS_URL` | ✅ | tham chiếu `${{Redis.REDIS_URL}}` tới service Redis của Railway (mạng nội bộ `redis.railway.internal:6379`) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
# Git Bash trên Windows làm hỏng mã hóa tiếng Việt trong -d (server trả 400
# "There was an error parsing the body") → gửi body từ file UTF-8:
#   printf '{"question":"Deploy là gì?"}' > ask.json
#   curl -i -X POST <URL>/ask ... --data-binary @ask.json

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
#    (dùng user riêng để lượt gọi ở bước 4 không chiếm quota)
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-ratelimit" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Output thật chạy ngày 2026-09-29 (Git Bash; `AGENT_API_KEY` lấy từ `DEPLOY_API_KEY` trong `.env`, không in giá trị):

```
$ curl -i <URL>/health
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 29 Sep 2026 04:22:40 GMT
Server: railway-hikari
x-railway-request-id: rHh-i2wYS2ehLnfe9fVATg
Content-Length: 57
x-hikari-trace: sin1.tr00
x-railway-edge: sin1
Connection: keep-alive

{"status":"ok","service":"day12-agent","version":"1.0.0"}

$ curl -i <URL>/ready
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 29 Sep 2026 04:22:41 GMT
Server: railway-hikari
x-railway-request-id: fXYeX_bOTMmgAZRGljLL4A
Content-Length: 31
x-hikari-trace: hkg1.aebn
x-railway-edge: hkg1
Connection: keep-alive

{"status":"ready","redis":true}

$ curl -i -X POST <URL>/ask (không có X-API-Key)
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Date: Tue, 29 Sep 2026 04:22:42 GMT
Server: railway-hikari
x-railway-request-id: ImkA6wD0Qq-u5meY2prcFg
Content-Length: 39
x-hikari-trace: hkg1.aebn
x-railway-edge: hkg1
Connection: keep-alive

{"detail":"invalid or missing API key"}

$ curl -i -X POST <URL>/ask -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv-test" --data-binary @ask.json
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 29 Sep 2026 04:23:03 GMT
Server: railway-hikari
x-railway-request-id: nMDpvhOQTEGmB4ngWUN5dQ
Content-Length: 279
x-hikari-trace: sin1.tr00
x-railway-edge: sin1
vary: accept-encoding
Connection: keep-alive

{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

$ for i in $(seq 1 15); do curl ... -H "X-User-Id: sv-ratelimit" ...; done
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
