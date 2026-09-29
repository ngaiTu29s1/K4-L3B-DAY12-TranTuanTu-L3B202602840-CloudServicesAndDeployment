# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trần Tuấn Tú |
| Mã học viên | L3B202602840 |
| Repo | https://github.com/ngaiTu29s1/K4-L3B-DAY12-TranTuanTu-L3B202602840-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://lab12.tutran-dev.id.vn |
| Platform | Self-hosted Server (Cloud Run / Koyeb architecture with Docker, Redis, Nginx Reverse Proxy & n8n CI/CD) |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Container expose nội bộ 8000, Nginx Proxy Manager forward traffic |
| `AGENT_API_KEY` | ✅ | Đặt trong file .env trên server host, không commit vào repo |
| `REDIS_URL` | ✅ | Redis container nội bộ `redis://day12-redis:6379/0` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://lab12.tutran-dev.id.vn/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://lab12.tutran-dev.id.vn/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://lab12.tutran-dev.id.vn/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://lab12.tutran-dev.id.vn/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://lab12.tutran-dev.id.vn/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness
HTTP/2 200 
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness
HTTP/2 200 
content-type: application/json
{"status":"ready","redis":true}

# 3. Không có API key (401)
HTTP/2 401 
content-type: application/json
{"detail":"invalid or missing API key"}

# 4. Có API key (200)
HTTP/2 200 
content-type: application/json
x-served-by: lab12.tutran-dev.id.vn
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

# 5. Rate limit (15 lần)
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Ghi Chú Kiến Trúc Triển Khai
Hệ thống được triển khai theo kiến trúc tự động hóa khép kín:
1. **GitHub Actions (CI)**: Tự động chạy Pytest, build multi-stage Docker image, push lên Docker Hub `ngaitu29s1/day12-agent:latest` và kích hoạt Webhook sang n8n khi merge/push vào nhánh `main`.
2. **n8n Workflow (CD)**: Lắng nghe Webhook bảo vệ bởi header `X-Deploy-Token`, SSH vào server để `docker compose pull agent` và `docker compose up -d agent`.
3. **Nginx Proxy Manager**: Đóng vai trò Reverse Proxy & SSL termination, định tuyến domain `lab12.tutran-dev.id.vn` trực tiếp vào container `day12-agent:8000` qua Docker network `homelab`.
