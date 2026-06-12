# Section 1 — Từ Localhost Đến Production

## Mục tiêu học
- Hiểu tại sao "it works on my machine" là vấn đề
- Nhận ra sự khác biệt giữa dev và production environment
- Áp dụng 4 nguyên tắc 12-factor cơ bản

---

## Ví dụ Basic — Agent "Kiểu Localhost"

```
develop/
├── app.py          # ❌ Anti-patterns: hardcode secrets, no config, no health check
├── .env.example
└── requirements.txt
```

### Chạy thử
```bash
cd basic
pip install -r requirements.txt
python app.py
# Truy cập: http://localhost:8000
```

### Những vấn đề trong code này:
1. API key hardcode trong code
2. Không có health check endpoint
3. Debug mode bật cứng
4. Không xử lý SIGTERM gracefully
5. Config không đến từ environment

---

## Ví dụ Advanced — 12-Factor Compliant Agent

```
production/
├── app.py          # ✅ Clean: config from env, health check, graceful shutdown
├── config.py       # ✅ Centralized config management
├── .env.example    # ✅ Template — không commit .env thật
└── requirements.txt
```

### Chạy thử
```bash
cd advanced
pip install -r requirements.txt
cp .env.example .env
# Sửa .env nếu cần
python app.py
```

### So sánh với Basic:

| | Basic (❌) | Advanced (✅) |
|--|-----------|--------------|
| Config | Hardcode trong code | Đọc từ env vars |
| Secrets | `api_key = "sk-abc123"` | `os.getenv("OPENAI_API_KEY")` |
| Port | Cố định `8000` | Từ `PORT` env var |
| Health check | Không có | `GET /health` |
| Shutdown | Tắt đột ngột | Graceful — hoàn thành request hiện tại |
| Logging | `print()` | Structured JSON logging |

---

## Câu hỏi thảo luận

1. **Điều gì xảy ra nếu bạn push code với API key hardcode lên GitHub public?**

   - Bot tự động scan GitHub liên tục để tìm các pattern key (`sk-...`, `AKIA...`...) — key bị lấy trong vài giây đến vài phút.
   - Key bị lạm dụng để gọi API tràn lan → hóa đơn hàng nghìn USD trong vài giờ.
   - Xóa file khỏi code chưa đủ — key vẫn còn trong **git history**. Phải dùng `git filter-repo`/BFG để xóa lịch sử **và** revoke + tạo key mới ngay lập tức.
   - Đây là lý do `production/config.py` đọc `OPENAI_API_KEY` từ `os.getenv(...)`, và `.env` (chứa secret thật) bị `.gitignore`, chỉ commit `.env.example` làm template.

2. **Tại sao stateless quan trọng khi scale?**

   - Stateless = app không lưu trạng thái (session, cache, file tạm...) trong memory/disk riêng của instance.
   - Khi scale, nhiều instance chạy sau load balancer — nếu state nằm ở instance A, request sau có thể vào instance B và không thấy state đó → lỗi/mất dữ liệu.
   - Cloud platform có thể kill/restart container bất kỳ lúc nào (deploy, auto-scale, crash). State trong container đó sẽ mất.
   - `production/app.py` xử lý SIGTERM để **graceful shutdown** (hoàn thành request hiện tại trước khi tắt) và có `/health`, `/ready` để platform biết instance nào sống/sẵn sàng — giúp tự do scale up/down.
   - State thật cần lưu (session, data) phải đưa ra backing service bên ngoài (Redis, Postgres, S3...).

3. **12-factor nói "dev/prod parity" — nghĩa là gì trong thực tế?**

   - Giảm khoảng cách giữa dev và production ở 3 khía cạnh: thời gian, người, và **công nghệ/cấu hình**.
   - Cùng 1 codebase (`production/app.py`) chạy cho cả dev và prod — khác biệt chỉ ở **giá trị biến môi trường** (`ENVIRONMENT`, `DEBUG`), không phải code khác nhau.
   - `config.py` có `validate()` raise lỗi ngay khi khởi động nếu thiếu `AGENT_API_KEY` ở production — "fail fast", phát hiện thiếu config trước khi deploy.
   - Anti-pattern ở `develop/app.py` (host `localhost`, port cứng, `reload=True`) vi phạm parity — chỉ chạy đúng trên máy dev, không chạy được trong container/cloud.
   - Parity cũng áp dụng cho backing services: nếu dev dùng SQLite còn prod dùng Postgres, bug liên quan DB sẽ không xuất hiện ở dev — chỉ nổ ra ở production.
