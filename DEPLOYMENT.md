# Thông Tin Deploy — Checkpoint 5

Service đã được triển khai trên Railway bằng Dockerfile trong repository. Các
secret chỉ được đặt trong Railway Variables; tài liệu này chỉ ghi tên biến,
không ghi giá trị.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyen Van Hong |
| Mã học viên | 2A202602800 |
| Repo | https://github.com/hongneuk65/K4-L3B-DAY12-NguyenVanHong-2A202602800-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-1408.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |
| Web service | `agent` — Dockerfile deployment |
| Redis service | `redis-runtime` — `redis:7-alpine`, private network |

## Biến Môi Trường Đã Set Trên Cloud

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán; app bind `0.0.0.0` |
| `AGENT_API_KEY` | ✅ | Secret trong Railway Variables, không nằm trong repo |
| `REDIS_URL` | ✅ | Private Redis service `redis-runtime` trong Railway |
| `RATE_LIMIT_PER_MINUTE` | ✅ | `10` |
| `MONTHLY_BUDGET_USD` | ✅ | `10.0` |
| `LOG_LEVEL` | ✅ | `INFO` |

## Lệnh Kiểm Tra

```bash
URL=https://agent-production-1408.up.railway.app

# 1. Liveness
curl -i "$URL/health"

# 2. Readiness — đã kết nối được Redis
curl -i "$URL/ready"

# 3. Không có API key
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — lấy AGENT_API_KEY từ secret store, không ghi vào repo
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: cp5-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Kiểm tra giới hạn 10 request/phút cho cùng user
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST "$URL/ask" \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: cp5-test" \
    -d '{"question":"test"}'
done
echo
```

## Kết Quả Chạy Thật

Đã kiểm tra trực tiếp URL public bằng HTTP client:

```text
GET /health
HTTP 200
{"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready
HTTP 200
{"status":"ready","redis":true}

POST /ask không có API key
HTTP 401
{"detail":"invalid or missing API key"}

POST /ask có API key hợp lệ
HTTP 200
response có answer, user_id=cp5-test và thông tin token/cost
```

Railway runtime đã xác nhận:

```text
agent: SUCCESS, 1 running replica, 0 crashed
redis-runtime: SUCCESS, 1 running replica, 0 crashed
```

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png` — bằng chứng trạng thái service Railway và public URL từ CLI.
- `screenshots/health.png` — kết quả `/health`, `/ready` và `/ask` không có key.

Các ảnh không chứa API key, Redis password hoặc giá trị secret.

## Ghi Chú Bảo Mật

- `AGENT_API_KEY` và thông tin xác thực Redis không được commit.
- `.env` nằm trong `.gitignore` và không được Git track.
- Railway tự cấp `PORT`; Dockerfile đọc biến này thay vì ghi đè bằng port cố định.
