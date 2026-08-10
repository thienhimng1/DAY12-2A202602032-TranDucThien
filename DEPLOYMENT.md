# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trần Đức Thiện |
| Mã học viên | 2A202602032 |
| Repo | https://github.com/thienhimng1/DAY12-2A202602032-TranDucThien |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-dc4a.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và nguồn giá trị, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis Add-on dịch vụ của platform |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-production-dc4a.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"}
curl -i https://day12-agent-production-dc4a.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-production-dc4a.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-production-dc4a.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

```json
{"status":"ok","service":"day12-agent","version":"1.0.0"}
{"status":"ready","redis":true}
{"detail":"invalid or missing API key"}
{"answer":"Deploy là quá trình đưa ứng dụng từ môi trường phát triển lên hạ tầng máy chủ.","user_id":"sv-test","history_length":0,"cost_usd":0.0001,"tokens":{"in":12,"out":35}}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
