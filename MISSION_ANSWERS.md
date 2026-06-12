# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

Tìm được trong `01-localhost-vs-production/develop/app.py`:

1. **API key hardcode trong code** (line 17): `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` — nếu push lên GitHub public, key bị lộ ngay và có thể bị lấy bởi bot scan.
2. **Database credentials hardcode** (line 18): `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"` — lộ luôn username/password của DB.
3. **Debug mode bật cứng** (line 21): `DEBUG = True` — không thể tắt debug khi chạy production mà không sửa code.
4. **Dùng `print()` thay vì structured logging, và log cả secret** (line 33-34, 38): `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` — secret bị in ra log, log không có format chuẩn để parse.
5. **Không có health check endpoint** (sau line 39) — platform (Railway, K8s...) không có cách biết agent còn sống để tự restart khi crash.
6. **Port cố định, không đọc từ env** (line 52): `port=8000` — trên cloud, platform inject `PORT` qua biến môi trường, hardcode port sẽ fail.
7. **Bind `host="localhost"`** (line 51) — chỉ nhận kết nối từ máy local, không chạy được trong container (cần `0.0.0.0`).
8. **`reload=True`** (line 53) — chế độ auto-reload dành cho dev, không nên bật trong production (tốn resource, không ổn định).
9. **Không xử lý SIGTERM / graceful shutdown** — khi container bị kill (deploy mới, scale down), request đang xử lý bị ngắt giữa chừng, mất dữ liệu.

### Exercise 1.2: Chạy basic version — Quan sát

App chạy được và trả lời request `/ask` bình thường, nhưng **không production-ready**, vì:

- Secrets (API key, DB password) hardcode trong code — nguy hiểm nếu push lên git public.
- Không có `/health` → platform không biết khi nào agent crash để tự restart.
- `DEBUG=True`, `reload=True` → không phù hợp production, tốn resource và có thể lộ thông tin debug.
- Log bằng `print()` và lộ cả API key ra console — không an toàn, không dùng được với log aggregator.
- Bind `host="localhost"` + port cứng `8000` → không chạy được trong container/cloud (platform inject `PORT` riêng và cần bind `0.0.0.0`).
- Không xử lý `SIGTERM` → mất request đang xử lý khi container bị kill/redeploy.

→ Kết luận: "Chạy được trên máy mình" ≠ "Sẵn sàng cho production" — cần áp dụng 12-factor (config qua env, health check, structured logging, graceful shutdown) như ở `production/app.py`.

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config | Hardcode trong code (`OPENAI_API_KEY`, `DATABASE_URL`, `DEBUG`, `MAX_TOKENS`) | Đọc từ env vars qua `config.py` (`Settings` dataclass + `os.getenv`, có `validate()` fail-fast) | Đổi config giữa dev/staging/prod không cần sửa code; secrets không nằm trong source code / git history |
| Secrets | `api_key = "sk-..."` hardcode, log ra cả console | `os.getenv("OPENAI_API_KEY")`, không bao giờ log secret | Tránh lộ key khi push code public hoặc khi đọc log |
| Health check | Không có | `/health` (liveness) + `/ready` (readiness) + `/metrics` | Platform dựa vào đây để biết container còn sống/sẵn sàng nhận traffic và tự động restart khi fail |
| Logging | `print()`, không có cấu trúc, log cả secret | Structured JSON logging qua module `logging`, chỉ log `question_length`, `client_ip`... | JSON logs dễ parse bởi log aggregator (Datadog, Loki); không log secret |
| Shutdown | Đột ngột — không bắt SIGTERM, `reload=True` | Graceful — `handle_sigterm` + `lifespan` context manager, hoàn thành request hiện tại trước khi tắt | Tránh mất request đang xử lý khi container bị kill (deploy mới, scale down) |
| Host/Port | `host="localhost"`, `port=8000` cố định | `host=settings.host` (`0.0.0.0`), `port=settings.port` từ `PORT` env var | Container phải bind `0.0.0.0` để nhận traffic từ ngoài; cloud platform tự inject `PORT` mỗi lần deploy |

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (dựa trên `02-docker/develop/Dockerfile`)

1. **Base image**: `python:3.11` — bản full (~1GB), có sẵn compiler/build tools, không tối ưu size.
2. **Working directory**: `/app` (`WORKDIR /app`).
3. **Tại sao `COPY requirements.txt .` rồi `RUN pip install` TRƯỚC khi `COPY app.py .`?**
   Docker build theo từng layer và cache lại mỗi layer. Nếu input của 1 layer không đổi so với lần build trước, Docker tái sử dụng cache, không chạy lại lệnh đó.
   - `requirements.txt` ít thay đổi, còn source code (`app.py`) thay đổi liên tục.
   - Đặt `COPY requirements.txt .` + `RUN pip install` trước → khi chỉ sửa code, layer `pip install` (tốn thời gian download + compile) vẫn được cache, chỉ layer copy code phía sau bị rebuild.
   - Nếu đảo ngược (`COPY . .` trước rồi mới `pip install`), bất kỳ thay đổi code nào — dù chỉ 1 dòng — cũng làm invalidate cache của `pip install`, buộc cài lại toàn bộ dependencies mỗi lần build → rất chậm.
4. **CMD vs ENTRYPOINT khác nhau thế nào?**
   - `CMD ["python", "app.py"]`: là lệnh **mặc định**, nhưng có thể bị **override hoàn toàn** bởi argument truyền vào `docker run` (ví dụ `docker run agent-develop python -c "print(1)"` sẽ chạy lệnh mới, bỏ qua CMD).
   - `ENTRYPOINT`: là lệnh **cố định luôn chạy**; argument từ `docker run` chỉ được **append** vào sau ENTRYPOINT (không override, trừ khi dùng `--entrypoint`).
   - Cả `develop/Dockerfile` và `production/Dockerfile` đều chỉ dùng `CMD`, không có `ENTRYPOINT` → vẫn linh hoạt override khi cần debug (`docker run -it agent-develop /bin/bash`).

### Exercise 2.2: Build và run (single-stage)

```bash
cd ../..   # về project root
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
docker run -p 8000:8000 my-agent:develop

curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'

docker images my-agent:develop
```

- Kết quả: container chạy được, `/ask` trả về câu trả lời từ mock LLM (`from utils.mock_llm import ask`).
- Image size thực tế (`docker images my-agent:develop`): **1.66 GB** — lớn vì base `python:3.11` đầy đủ mang theo compiler, build tools, docs... không cần cho runtime.

### Exercise 2.3: Multi-stage build (dựa trên `02-docker/production/Dockerfile`)

- **Stage 1 — `builder`**:
  - Base `python:3.11-slim`, cài thêm `gcc`, `libpq-dev` (build tools cần để compile các package có native extension như psycopg2, numpy...).
  - `pip install --no-cache-dir --user -r requirements.txt` → cài packages vào `/root/.local` (tách riêng khỏi hệ thống, dễ copy sang stage sau).
  - Đây là image "tạm", chứa toàn bộ build toolchain, **không** dùng để deploy.

- **Stage 2 — `runtime`**:
  - Base `python:3.11-slim` MỚI (sạch, không kế thừa layer của builder).
  - Tạo non-root user `appuser` (security best practice).
  - `COPY --from=builder /root/.local /home/appuser/.local` → chỉ lấy site-packages đã cài sẵn, KHÔNG mang theo gcc/apt cache/build artifacts.
  - Copy source code (`main.py`, `utils/mock_llm.py`), chạy với `USER appuser`, có `HEALTHCHECK`.

- **Tại sao image cuối nhỏ hơn?**
  - `gcc`, `libpq-dev`, apt cache, pip download cache chỉ tồn tại trong layer của stage `builder` — stage này bị **loại bỏ hoàn toàn** khỏi image cuối; chỉ `/root/.local` (packages đã build) được copy qua.
  - Cả 2 stage dùng `python:3.11-slim` (~150MB) thay vì `python:3.11` full (~1GB) như ở `develop/Dockerfile`.
  - `--no-cache-dir` khi `pip install` → không giữ lại wheel cache.
  → Runtime image cuối chỉ chứa: Python interpreter + dependencies cần CHẠY + source code — không có compiler, build tools, cache.

### Exercise 2.3: Image size comparison
- Develop (`my-agent:develop`, single-stage `python:3.11`): **1660 MB**
- Production (`agent-production`, multi-stage `python:3.11-slim`): **236 MB** (đạt mục tiêu < 500MB)
- Difference: **~85.8%** nhỏ hơn (= (1660-236)/1660 × 100), tức giảm gần **7 lần**.

### Exercise 2.4: Docker Compose stack (dựa trên `02-docker/production/docker-compose.yml` + `nginx/nginx.conf`)

**Services được start** (network `internal`, driver `bridge`):
1. `agent` — FastAPI app, build từ Dockerfile (stage `runtime`), có thể scale nhiều replicas, **không** expose port ra host — chỉ truy cập được qua nginx.
2. `redis` — `redis:7-alpine`, cache session & rate limiting, `maxmemory 256mb` (LRU eviction).
3. `qdrant` — `qdrant/qdrant:v1.9.0`, vector database cho RAG.
4. `nginx` — `nginx:alpine`, expose `80`/`443` ra ngoài, đóng vai trò reverse proxy + load balancer + rate limiting.

**Architecture diagram**:
```
                  Client
                    │
                    ▼  :80 / :443
            ┌──────────────┐
            │     nginx     │  reverse proxy, LB, rate-limit (10 r/s, burst 20)
            └──────┬───────┘
                    │ proxy_pass → upstream agent_backend (agent:8000)
                    ▼
            ┌──────────────┐
            │    agent      │  FastAPI (scale ra nhiều replicas)
            └──┬────────┬──┘
               │        │
       REDIS_URL      QDRANT_URL
               │        │
               ▼        ▼
         ┌────────┐  ┌────────┐
         │ redis  │  │ qdrant │
         └────────┘  └────────┘

        (tất cả trong network "internal")
```

**Communication**:
- Tất cả service nằm trong network `internal` → gọi nhau qua **service name** (Docker DNS), ví dụ `agent` kết nối Redis qua `REDIS_URL=redis://redis:6379/0` và Qdrant qua `QDRANT_URL=http://qdrant:6333`.
- `nginx` là cổng vào duy nhất từ bên ngoài; `agent` không map port ra host, nên mọi traffic buộc phải đi qua nginx → nginx `proxy_pass` tới `upstream agent_backend { server agent:8000; }` (round-robin khi có nhiều replicas).
- `agent` chỉ start sau khi `redis` và `qdrant` báo `healthy` (`depends_on.condition: service_healthy`); `nginx` start sau `agent`.
- nginx áp rate limit 10 req/s/IP (burst 20, trả `429` khi vượt) — lớp bảo vệ đầu tiên trước khi request chạm vào agent.

**Test** (sau khi `docker compose up -d`):
```bash
curl http://localhost/health
curl -X POST http://localhost/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain microservices"}'
```

Kết quả thực tế:
```
$ docker compose ps
NAME                  IMAGE                  STATUS                  PORTS
production-agent-1    production-agent       Up (healthy)            8000/tcp
production-nginx-1    nginx:alpine           Up                      0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
production-qdrant-1   qdrant/qdrant:v1.9.0   Up (healthy)            6333-6334/tcp
production-redis-1    redis:7-alpine         Up (healthy)            6379/tcp

$ curl http://localhost/health
{"status":"ok","uptime_seconds":112.2,"version":"2.0.0","timestamp":"2026-06-12T10:10:33.798755"}

$ curl -X POST http://localhost/ask -H "Content-Type: application/json" -d '{"question": "Explain microservices"}'
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

→ Cả 4 service đều `Up (healthy)`. Request đi qua `nginx` (port 80) → `agent` → mock LLM, trả về đúng kết quả.

**Debug note (liên quan Checkpoint 2 — debug container)**:
Lần đầu `docker compose up`, `qdrant` bị stuck ở trạng thái `unhealthy` dù log cho thấy Qdrant đã start xong (`Qdrant HTTP listening on 6333`). Dùng `docker inspect production-qdrant-1 --format '{{json .State.Health}}'` thì thấy lỗi:
```
exec: "curl": executable file not found in $PATH
```
→ Image `qdrant/qdrant:v1.9.0` (Debian slim) không có `curl`, nên healthcheck `curl -f http://localhost:6333/health` luôn fail — không phải do Qdrant lỗi, mà do **tool kiểm tra không tồn tại trong image**. Vì `agent` có `depends_on: qdrant: condition: service_healthy`, agent không bao giờ start được.
Fix: đổi healthcheck sang dùng `bash` (có sẵn trong image) với `/dev/tcp` để check port mở:
```yaml
test: ["CMD", "bash", "-c", "exec 3<>/dev/tcp/127.0.0.1/6333"]
```

### Câu hỏi thảo luận (`02-docker/README.md`)

1. **Tại sao `COPY requirements.txt .` rồi `RUN pip install` TRƯỚC `COPY . .`?** → xem giải thích ở Exercise 2.1, câu 3 (layer caching).
2. **`.dockerignore` nên chứa gì? Tại sao `venv/` và `.env` quan trọng?**
   Theo `02-docker/develop/.dockerignore`: `__pycache__/`, `*.pyc/*.pyo/*.pyd`, `venv/`/`env/`/`.venv/`, `.vscode/`/`.idea/`, `.DS_Store`/`Thumbs.db`, `.env`/`.env.*` (trừ `.env.example`), `.git/`/`.gitignore`, `*.md`, `tests/`, `docs/`.
   - `venv/` quan trọng vì: (1) rất nặng (hàng trăm MB), làm build context lớn → build chậm; (2) chứa path/binary cố định của máy host, không tương thích với container (image tự `pip install` riêng dependencies của nó).
   - `.env` quan trọng vì chứa **secrets** (API key, DB password...). Nếu lọt vào build context và bị `COPY` vào image, bất kỳ ai `docker pull`/`docker history`/`docker save` image đều đọc được secret — kể cả khi sau đó có xoá file ở layer khác, layer cũ chứa secret vẫn nằm trong image history.
3. **Nếu agent cần đọc file từ disk, làm sao mount volume vào container?**
   Dùng `volumes:` trong `docker-compose.yml` hoặc `-v` với `docker run`:
   - Bind mount (map thư mục host vào container): `- ./data:/app/data`.
   - Named volume (Docker quản lý, persist qua các lần restart) — đã dùng trong compose này: `redis_data:/data` và `qdrant_data:/qdrant/storage`.
   - Với `docker run`: `docker run -v $(pwd)/data:/app/data agent-production`.

## Part 3: Cloud Deployment

> **Note:** Phần này giả định đã deploy thành công lên Railway theo `03-cloud-deployment/railway/` (chưa thực sự chạy `railway up` — không có Railway account/CLI trong môi trường làm bài). Các bước, config và kết quả mong đợi dưới đây dựa trên đọc `railway.toml`/`app.py` thực tế trong repo; URL là URL ví dụ theo đúng format Railway sinh ra.

### Exercise 3.1: Railway deployment (`03-cloud-deployment/railway/`)

**Config (`railway.toml`):**
- `builder = "NIXPACKS"` — Railway không cần Dockerfile, tự detect Python + `requirements.txt` và build.
- `startCommand = "uvicorn app:app --host 0.0.0.0 --port $PORT"` — khớp với `app.py:63-65` (`port = int(os.getenv("PORT", 8000))`, `host="0.0.0.0"`) → bắt buộc đọc `PORT` từ env vì Railway tự inject port khác nhau mỗi lần deploy.
- `healthcheckPath = "/health"`, `healthcheckTimeout = 30` — Railway gọi `GET /health` (`app.py:47-58`, trả `{"status":"ok","uptime_seconds":...,"platform":"Railway","timestamp":...}`); nếu fail/timeout 30s → container bị coi unhealthy.
- `restartPolicyType = "ON_FAILURE"`, `restartPolicyMaxRetries = 3` — crash thì Railway tự restart tối đa 3 lần trước khi đánh dấu deploy failed.

**Các bước đã thực hiện (giả định):**
```bash
cd 03-cloud-deployment/railway
railway login
railway init
railway variables set ENVIRONMENT=production
railway up
railway domain
```

- URL: `https://ai-agent-production.up.railway.app` *(URL ví dụ, theo format `*.up.railway.app` mà Railway sinh ra ở bước `railway domain`)*

**Test & kết quả mong đợi:**
```bash
$ curl https://ai-agent-production.up.railway.app/health
{"status":"ok","uptime_seconds":42.7,"platform":"Railway","timestamp":"2026-06-12T12:00:00.000000+00:00"}

$ curl -X POST https://ai-agent-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain microservices"}'
{"question":"Explain microservices","answer":"Đây là câu trả lời từ AI agent (mock)...","platform":"Railway"}
```

- Screenshot: [Link to screenshot in repo] *(chưa có — cần chạy thật để chụp dashboard Railway)*

### Exercise 3.2: Render deployment — so sánh `render.yaml` vs `railway.toml`

| | `railway.toml` | `render.yaml` |
|---|---|---|
| Cách khai báo | Config file cho **1 service** (app hiện tại), build/deploy command + policy | **Infrastructure as Code** cho **nhiều service** cùng lúc (`services:` là list) — ở đây gồm `ai-agent` (web) + `agent-cache` (Redis) |
| Build | `builder = "NIXPACKS"` — Railway tự detect | `buildCommand: pip install -r requirements.txt` — khai báo rõ ràng |
| Start command | `startCommand = "uvicorn app:app --host 0.0.0.0 --port $PORT"` | `startCommand: uvicorn app:app --host 0.0.0.0 --port $PORT` — **giống nhau**, cả 2 platform đều inject `$PORT` |
| Health check | `healthcheckPath = "/health"` + `healthcheckTimeout = 30` | `healthCheckPath: /health` — tương tự, không có timeout riêng |
| Region | Không khai báo trong file (chọn ở dashboard) | `region: singapore` — khai báo trực tiếp trong YAML |
| Plan/Pricing | Không khai báo (Railway dùng usage-based $5 credit) | `plan: free` — khai báo tier ngay trong code |
| Restart policy | `restartPolicyType = "ON_FAILURE"`, `restartPolicyMaxRetries = 3` — khai báo rõ | Không có field tương đương — Render tự quản lý restart |
| Secrets | Set qua `railway variables set ...` (CLI), không nằm trong file | `envVars:` khai báo **tên** biến trong file, nhưng giá trị thật set qua dashboard (`sync: false`) hoặc tự sinh (`generateValue: true` cho `AGENT_API_KEY`) |
| Dependencies khác | Không có — Redis/DB phải tạo riêng | Khai báo luôn `type: redis` (`agent-cache`) trong cùng file → "1 file = toàn bộ stack" |
| Auto deploy | Qua `railway up` (CLI) hoặc GitHub integration (cấu hình ở dashboard) | `autoDeploy: true` — khai báo ngay trong file, push GitHub là tự deploy |

→ **Khác biệt cốt lõi:** `railway.toml` tập trung vào *cách chạy 1 service* (build/start/health/restart), còn `render.yaml` là **bản thiết kế toàn bộ infrastructure** (nhiều service + dependencies + region + plan) commit vào git — đúng tinh thần "Infrastructure as Code". Railway thiên về tốc độ (gõ `railway up` là xong), Render thiên về khai báo tường minh, dễ review trong PR.

### Exercise 3.3 (Optional): GCP Cloud Run — `cloudbuild.yaml` + `service.yaml`

**`cloudbuild.yaml` — CI/CD pipeline (chạy khi push lên `main`):**
1. **`test`**: cài `requirements.txt` + `pytest`, chạy `pytest tests/ -v` — **build sẽ fail sớm nếu test fail**, không tốn công build/push image lỗi.
2. **`build`** (chờ `test`): `docker build` với 2 tag (`$COMMIT_SHA` và `latest`), dùng `--cache-from=...:latest` để tái sử dụng layer cache giữa các lần build → build nhanh hơn.
3. **`push`** (chờ `build`): push cả 2 tag lên Container Registry (`gcr.io/$PROJECT_ID/ai-agent`).
4. **`deploy`** (chờ `push`): `gcloud run deploy` với:
   - `--allow-unauthenticated` — endpoint public (auth tự xử lý ở tầng app, như Part 4).
   - `--min-instances=1` — giữ tối thiểu 1 instance **luôn chạy** để tránh cold start (đánh đổi: tốn phí dù idle).
   - `--max-instances=10` — chặn chi phí khi traffic spike đột biến.
   - `--memory=512Mi --cpu=1 --timeout=60s` — giới hạn resource/request timeout.
   - `--set-secrets=OPENAI_API_KEY=openai-key:latest` — secret lấy từ **Secret Manager**, không nằm trong env vars thường (khác hẳn cách hardcode ở `01-localhost-vs-production/develop`).
- `options.machineType: E2_MEDIUM` + `timeout: 1200s` — giới hạn máy build và tổng thời gian pipeline (20 phút).

**`service.yaml` — Cloud Run service definition (declarative, dùng `gcloud run services replace`):**
- `autoscaling.knative.dev/minScale: "1"` / `maxScale: "10"` — khớp với `cloudbuild.yaml`, lặp lại config scaling dưới dạng YAML để có thể version-control & review riêng khỏi pipeline.
- `containerConcurrency: 80` + `autoscaling.../target: "80"` — mỗi instance xử lý tối đa 80 request đồng thời; Cloud Run scale thêm instance khi vượt 80 — phù hợp với agent có I/O-bound (chờ LLM API) vì 1 instance vẫn xử lý được nhiều request song song.
- **2 loại probe khác nhau** (liên kết Part 5):
  - `livenessProbe` → `GET /health`, `initialDelaySeconds: 10`, `periodSeconds: 30` — kiểm tra định kỳ, restart nếu fail.
  - `startupProbe` → `GET /ready`, `periodSeconds: 3`, `failureThreshold: 10` (tối đa ~30s để app sẵn sàng lần đầu) — trong lúc startup probe chưa pass, traffic **không** được route vào, và liveness probe **chưa** chạy (tránh restart loop khi app chỉ đang khởi động chậm).
- Secrets (`OPENAI_API_KEY`, `AGENT_API_KEY`) dùng `valueFrom.secretKeyRef` trỏ tới Secret Manager — không bao giờ xuất hiện plaintext trong `service.yaml` (file này commit được vào git an toàn).

→ So với Railway/Render (Tier 1 — "deploy < 10 phút, không cần config nhiều"), Cloud Run (Tier 2) đòi hỏi tự khai báo CI/CD pipeline + scaling + probes + secrets management chi tiết hơn, đổi lại kiểm soát production tốt hơn (đúng bảng "3 Tier" ở `03-cloud-deployment/README.md`).

### Câu hỏi thảo luận (`03-cloud-deployment/README.md`)

1. **Tại sao serverless (Lambda) không phải lúc nào cũng tốt cho AI agent?**
   - **Cold start nặng**: package AI agent thường có dependencies lớn (model libs, vector DB client...) → cold start của Lambda có thể mất nhiều giây, tệ hơn nhiều so với 1 container đã warm.
   - **Timeout giới hạn** (Lambda tối đa 15 phút, nhiều platform để mặc định 30s-60s) — LLM call có thể chậm, đặc biệt với streaming hoặc multi-step agent (tool calling nhiều round).
   - **Stateless triệt để + không có long-lived connection** — khó giữ WebSocket/streaming response, khó pool connection tới Redis/DB hiệu quả (mỗi invocation có thể là cold connection mới).
   - **Cost model theo số request/thời gian chạy** — với traffic đều đặn (không spike-y), 1 instance Cloud Run/Railway chạy liên tục (`min-instances=1`) có thể rẻ hơn hàng nghìn lambda invocation ngắn.
   → Lambda phù hợp cho workload event-driven, ngắn, traffic không đều; container (Cloud Run/Railway) phù hợp hơn cho AI agent cần giữ warm + xử lý request lâu hơn.

2. **"Cold start" là gì? Ảnh hưởng thế nào đến UX?**
   - Cold start = thời gian platform phải **khởi tạo instance mới** (pull image, start container, load model/dependencies, chạy `lifespan` startup) trước khi xử lý được request đầu tiên — xảy ra khi: instance đầu tiên, hoặc scale từ 0 → 1, hoặc sau thời gian idle bị platform tắt instance.
   - Ảnh hưởng UX: request đầu tiên (hoặc request ngay sau khi idle lâu) có latency tăng đột biến — có thể từ vài trăm ms đến vài giây (image nặng như `develop` 1.66GB ở Part 2 sẽ cold start chậm hơn nhiều so với `production` 236MB). Với agent realtime/chat, user sẽ thấy "đứng" vài giây ở lần gõ đầu tiên.
   - Giảm cold start: image nhỏ (multi-stage — Part 2), `min-instances=1` (Cloud Run `service.yaml`) hoặc Railway/Render giữ instance luôn chạy (trade-off: tốn phí idle).

3. **Khi nào nên upgrade từ Railway lên Cloud Run?**
   - Khi cần **kiểm soát scaling chi tiết**: `minScale`/`maxScale`/`containerConcurrency` (Railway không cho config mức này).
   - Khi cần **CI/CD pipeline tường minh** với test gate trước khi deploy (`cloudbuild.yaml` step `test` — Railway không chạy test trước deploy).
   - Khi cần **secrets management** chuẩn (Secret Manager + `secretKeyRef`) thay vì set qua dashboard/CLI biến môi trường thường.
   - Khi traffic đủ lớn để chi phí usage-based của Railway ($5 credit/tháng) không còn đủ, hoặc cần SLA/region control chặt hơn (region cố định `asia-southeast1`, multi-region).
   - Ngược lại: nếu vẫn là MVP/demo, ít traffic, không cần CI/CD phức tạp → Railway/Render (Tier 1) vẫn là lựa chọn nhanh và rẻ hơn, đúng tinh thần bảng so sánh 3-tier ở đầu `03-cloud-deployment/README.md`.

## Part 4: API Security

### Exercise 4.1: API Key authentication (`04-api-gateway/develop/app.py`)

- **API key được check ở đâu?** Trong dependency `verify_api_key()` (`app.py:39-54`), được inject vào endpoint `/ask` qua `Depends(verify_api_key)` (`app.py:70`). Header `X-API-Key` được đọc bằng `APIKeyHeader(name="X-API-Key", auto_error=False)` (`app.py:36`) và so sánh với `API_KEY = os.getenv("AGENT_API_KEY", "demo-key-change-in-production")` (`app.py:35`).
- **Điều gì xảy ra nếu sai key?**
  - Thiếu header `X-API-Key` → `401 Missing API key. Include header: X-API-Key: <your-key>` (`app.py:45-48`).
  - Có header nhưng giá trị sai → `403 Invalid API key.` (`app.py:50-53`).
  - `/` và `/health` không yêu cầu key (public), chỉ `/ask` bị bảo vệ.
- **Làm sao rotate key?** Đổi giá trị env `AGENT_API_KEY` rồi restart service — không cần sửa code (key chỉ so sánh với 1 biến env, không lưu DB). Hạn chế: toàn bộ client dùng **chung 1 key**, nên rotate sẽ làm mọi client cũ mất quyền truy cập cùng lúc — không revoke được riêng từng client. Đây chính là lý do production chuyển sang JWT (Exercise 4.2): mỗi user có token riêng, revoke/expire độc lập.

### Exercise 4.2: JWT authentication (`04-api-gateway/production/auth.py`)

**Flow:**
1. Client `POST /auth/token` với `{username, password}` → `authenticate_user()` (`auth.py:70-75`) kiểm tra trong `DEMO_USERS` (`auth.py:27-30`):
   - `student` / `demo123` → role `user`, `daily_limit: 50`
   - `teacher` / `teach456` → role `admin`, `daily_limit: 1000`
   - Sai username/password → `401 Invalid credentials`.
2. Nếu hợp lệ, `create_token()` (`auth.py:35-43`) tạo JWT với payload `{sub: username, role, iat, exp}`, `exp = now + 60 phút`, ký bằng `SECRET_KEY` (env `JWT_SECRET`, default `super-secret-change-in-production-please`) thuật toán `HS256`. Trả về `{access_token, token_type: "bearer", expires_in_minutes: 60}`.
3. Các request sau gửi `Authorization: Bearer <token>` → `verify_token()` (`auth.py:46-67`) decode + verify signature, trả `{username, role}`:
   - Thiếu header → `401` (`headers: {"WWW-Authenticate": "Bearer"}`)
   - Token hết hạn → `401 Token expired. Please login again.`
   - Token sai signature/malformed → `403 Invalid token.`

→ JWT là **stateless**: server không cần lưu session ở đâu, chỉ cần verify signature mỗi request — phù hợp khi scale nhiều instance (liên kết Part 5).

### Exercise 4.3: Rate limiting (`04-api-gateway/production/rate_limiter.py`)

- **Algorithm:** Sliding Window Counter (không phải token bucket). Mỗi user có 1 `deque` timestamps (`rate_limiter.py:27`). Mỗi request: xoá các timestamp cũ hơn `window_seconds` (`rate_limiter.py:39-40`), nếu số timestamp còn lại `>= max_requests` → raise `429` kèm headers `X-RateLimit-Limit/Remaining/Reset` + `Retry-After` (`rate_limiter.py:45-62`); nếu chưa vượt → append timestamp hiện tại và cho qua.
- **Limit:**
  | Tier | Limit | Khởi tạo |
  |---|---|---|
  | User thường | 10 req/phút | `rate_limiter_user = RateLimiter(max_requests=10, window_seconds=60)` (`rate_limiter.py:86`) |
  | Admin | 100 req/phút | `rate_limiter_admin = RateLimiter(max_requests=100, window_seconds=60)` (`rate_limiter.py:87`) |
- **Bypass cho admin:** Không phải "bypass" tuyệt đối — `production/app.py:140` chọn limiter theo role: `limiter = rate_limiter_admin if role == "admin" else rate_limiter_user`. Admin vẫn bị giới hạn, chỉ là quota cao hơn (100 vs 10 req/phút) nhờ token JWT chứa `role` (lấy từ `DEMO_USERS`).
- **Quan sát khi hit limit:** gọi liên tục >10 lần/phút với token `student` → từ request thứ 11 trả `429` với body `{"error": "Rate limit exceeded", "limit": 10, "window_seconds": 60, "retry_after_seconds": ...}` và header `Retry-After`.

### Exercise 4.4: Cost guard (`04-api-gateway/production/cost_guard.py`)

CODE_LAB.md đưa ra TODO stub đơn giản (Redis, budget $10/**tháng**, key `budget:{user_id}:{YYYY-MM}`). Code thực tế trong repo **đã implement sẵn**, đầy đủ và chi tiết hơn:

1. **Pricing theo token** (`cost_guard.py:20-21`): `PRICE_PER_1K_INPUT_TOKENS = 0.00015`, `PRICE_PER_1K_OUTPUT_TOKENS = 0.0006` (giá GPT-4o-mini). `UsageRecord.total_cost_usd` (`cost_guard.py:32-36`) tính cost = `input_tokens/1000 * price_in + output_tokens/1000 * price_out`.
2. **2 mức budget** (`cost_guard.py:40-51`, khởi tạo ở `cost_guard.py:128`): `daily_budget_usd=1.0` (mỗi user/ngày) và `global_daily_budget_usd=10.0` (toàn hệ thống/ngày), `warn_at_pct=0.8`.
3. **`check_budget(user_id)`** (`cost_guard.py:60-91`), gọi **trước** khi LLM xử lý (`production/app.py:144`):
   - Global budget vượt → `503 Service temporarily unavailable...` (chặn **toàn bộ** user, không riêng ai).
   - Per-user budget vượt → `402 Payment Required` kèm `{used_usd, budget_usd, resets_at: "midnight UTC"}`.
   - Đã dùng ≥80% → `logger.warning(...)`.
4. **`record_usage(user_id, input_tokens, output_tokens)`** (`cost_guard.py:93-110`), gọi **sau** khi có response (`production/app.py:150-152`): cộng token vào `UsageRecord` của user và vào `_global_cost`.
5. **`get_usage(user_id)`** (`cost_guard.py:112-124`) → phục vụ endpoint `GET /me/usage`.

**So sánh với solution gợi ý trong CODE_LAB.md:**

| | Solution CODE_LAB.md | Code thực tế (`cost_guard.py`) |
|---|---|---|
| Storage | Redis, key theo tháng | In-memory dict, theo ngày (reset tự động khi `record.day != today`, `cost_guard.py:56`) |
| Đơn vị budget | $/tháng, 1 mức (per-user) | $/ngày, **2 mức**: per-user ($1) + global ($10) |
| Tính cost | `estimated_cost` truyền sẵn vào | Tính từ token thực tế (input/output riêng, giá khác nhau) |
| Cảnh báo sớm | Không có | Có (`warn_at_pct=0.8`) |

**Hạn chế:** in-memory (`self._records`, `self._global_cost`) → mất hết khi restart, và khi scale nhiều instance (Part 5) mỗi instance có budget riêng → user có thể "lách" budget bằng cách rotate qua nhiều instance. Cần chuyển sang Redis (giống cách `production/app.py` ở Part 5 lưu session) để budget được chia sẻ đúng giữa các instance.

### Câu hỏi thảo luận (`04-api-gateway/README.md`)

1. **Khi nào nên dùng API Key vs JWT vs OAuth2?**
   - **API Key**: server-to-server, B2B, internal tools, MVP — đơn giản, không cần định danh user, dễ rotate qua env var nhưng không revoke được riêng từng client (1 key dùng chung).
   - **JWT**: app có user login, cần role/permission (user/admin), muốn stateless để scale nhiều instance không cần session store dùng chung — token tự chứa thông tin + tự expire (60 phút như ở `auth.py`).
   - **OAuth2**: cần đăng nhập qua bên thứ 3 (Google/GitHub login), hoặc nhiều client/scope khác nhau truy cập tài nguyên của nhiều user — phức tạp hơn nhưng chuẩn hoá, có refresh token, scope-based permission.

2. **Rate limit nên đặt bao nhiêu request/phút cho một AI agent?**
   Phụ thuộc cost/latency của model phía sau. Repo này chọn **10 req/phút cho user, 100 req/phút cho admin** — hợp lý cho demo/mock. Với LLM thật (vài giây/request, tính tiền theo token), nên đặt thấp hơn cho free tier (5-10 req/phút) và kết hợp `cost_guard` (giới hạn theo **$**) — vì rate limit chỉ chặn được *tốc độ* spam, không chặn được *tổng chi phí* nếu mỗi request gửi câu hỏi rất dài.

3. **Nếu API key bị lộ, bạn phát hiện và xử lý như thế nào?**
   - **Phát hiện:** theo dõi usage bất thường — spike số request trong `rate_limiter.get_stats()` hoặc cost tăng vọt trong `cost_guard.get_usage()`/`_global_cost`, request từ IP/giờ lạ trong log.
   - **Xử lý:**
     - Với API Key (4.1): đổi `AGENT_API_KEY` + restart → key cũ vô hiệu **ngay lập tức** (so sánh trực tiếp với env var, không cache).
     - Với JWT (4.2): đổi `JWT_SECRET` → mọi token cũ (ký bằng secret cũ) lập tức `403 Invalid token` ở `verify_token()`. Vì token chỉ sống 60 phút (`ACCESS_TOKEN_EXPIRE_MINUTES`), thời gian "khai thác" tối đa của 1 token bị lộ cũng bị giới hạn sẵn.
     - Sau đó: bắt user đổi lại password/credentials, audit log để xác định phạm vi đã bị lợi dụng (số request, cost đã phát sinh qua `cost_guard`).

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks (`05-scaling-reliability/develop/app.py`)

- **`GET /health`** (`app.py:104-144`) — *Liveness probe*: trả `200` gần như luôn luôn (chỉ phản ánh "process còn chạy"), gồm `status`, `uptime_seconds`, `version`, `environment`, `timestamp`, và `checks` (hiện tại chỉ check RAM qua `psutil` nếu có cài, optional — `app.py:122-131`). Platform (Railway/K8s) gọi định kỳ; nếu fail/timeout → **restart container**.
- **`GET /ready`** (`app.py:147-168`) — *Readiness probe*: dựa vào biến global `_is_ready` (set `True` sau khi "load model" xong trong `lifespan` startup, `app.py:41-50`; set `False` khi shutdown, `app.py:55`). Chưa ready hoặc đang shutdown → `503 Agent not ready...`; ready → `{"ready": true, "in_flight_requests": N}`. Load balancer dùng endpoint này để quyết định **có route traffic vào instance này hay không**.
- **Khác biệt:** `/health` = "tôi còn sống" (quyết định restart), `/ready` = "tôi sẵn sàng nhận request" (quyết định routing) — 2 khái niệm tách biệt dù có thể cùng trả 200 trong trạng thái bình thường.

### Exercise 5.2: Graceful shutdown (`05-scaling-reliability/develop/app.py`)

Cơ chế gồm 3 phần phối hợp:

1. **Đếm request đang chạy** — middleware `track_requests` (`app.py:72-81`): tăng `_in_flight_requests` khi request vào, giảm trong `finally` (đảm bảo giảm dù request lỗi).
2. **`lifespan` shutdown** (`app.py:54-66`): khi nhận SIGTERM, uvicorn trigger phần shutdown của `lifespan` →
   - Set `_is_ready = False` ngay → `/ready` trả `503` → load balancer **dừng route request mới** vào instance này.
   - Loop chờ tối đa **30s** cho `_in_flight_requests` về 0 (`while _in_flight_requests > 0 and elapsed < timeout`).
   - Log `"✅ Shutdown complete"`.
3. **`handle_sigterm`/SIGINT** (`app.py:175-187`) chỉ log thêm — uvicorn tự bắt SIGTERM và gọi lifespan shutdown; `timeout_graceful_shutdown=30` trong `uvicorn.run()` (`app.py:198`) cho uvicorn tối đa 30s hoàn tất request hiện tại trước khi force-kill.

**Kết quả test (theo code, mô tả luồng):** request đang xử lý lúc `kill -TERM $PID` → vẫn nhận được response đầy đủ (vì `_in_flight_requests > 0` nên shutdown chờ); bất kỳ request mới gửi sau đó → `503 Agent not ready` (vì `_is_ready=False`) cho đến khi process thực sự thoát.

### Exercise 5.3: Stateless design (`05-scaling-reliability/production/app.py`)

File này chính là phiên bản đã refactor đúng theo "Correct" pattern của CODE_LAB.md (thay vì `conversation_history = {}` trong memory):

- **Session lưu ở Redis**, có TTL: `save_session`/`load_session`/`append_to_history` (`app.py:59-90`) — `_redis.setex(f"session:{session_id}", 3600, json.dumps(data))`.
- **Fallback an toàn**: nếu kết nối Redis lúc start thất bại (`app.py:35-45`), tự chuyển sang `_memory_store: dict` và in cảnh báo `"⚠️ Redis not available — using in-memory store (not scalable!)"` — code tự document rõ đây là chế độ **không scale được**.
- **`POST /chat`** (`app.py:128-157`): nhận `{question, session_id?}` — `session_id=None` thì tạo `uuid4` mới; append câu hỏi + câu trả lời vào history trong Redis; response trả kèm `served_by: INSTANCE_ID` (`app.py:155`, `INSTANCE_ID` lấy từ env `INSTANCE_ID` hoặc random hex) — đây là field dùng để **chứng minh** bất kỳ instance nào cũng đọc/viết được session chung.
- **`GET /chat/{session_id}/history`** và **`DELETE /chat/{session_id}`**: xem/xoá session — hoạt động đúng dù request này rơi vào instance khác với instance đã tạo session, vì state nằm ở Redis chung, không nằm trong memory của 1 instance.
- **`/health`, `/ready`** (`app.py:187-215`) đều gọi `_redis.ping()` — Redis chết thì `/health` báo `degraded`, `/ready` trả `503` (đúng vai trò readiness: dependency chưa sẵn sàng → ngừng nhận traffic).

### Exercise 5.4: Load balancing (`05-scaling-reliability/production/docker-compose.yml` + `nginx.conf`)

**Kiến trúc:**
```
Client → nginx (:8080) → agent_cluster (3 replicas, round-robin qua Docker DNS) → redis (session chung)
```

- `agent` service có `deploy.replicas: 3` (`docker-compose.yml:42`) — chạy `docker compose up --scale agent=3` sẽ tạo 3 container `agent`, **không expose port ra host**, chỉ giao tiếp qua network `agent_net`.
- `nginx.conf`: `resolver 127.0.0.11 valid=10s` (DNS nội bộ của Docker) + `upstream agent_cluster { server agent:8000; keepalive 16; }` (`nginx.conf:7-11`) — Docker DNS trả về địa chỉ của **cả 3 replica** dưới cùng tên `agent`, nginx round-robin giữa chúng.
- `add_header X-Served-By $upstream_addr always;` (`nginx.conf:17`) — mỗi response có header cho biết request được forward tới IP container nào → cách trực quan để quan sát load balancing (bổ sung cho field `served_by` ở tầng app).
- `proxy_next_upstream error timeout http_503` + `proxy_next_upstream_tries 3` (`nginx.conf:23-24`) — nếu 1 instance lỗi/timeout/503, nginx tự retry sang instance khác (failover) tối đa 3 lần.
- `agent` có `depends_on: redis: condition: service_healthy` (`docker-compose.yml:28-30`) — đảm bảo Redis sẵn sàng trước khi agent start, vì agent cần Redis để lưu session (Exercise 5.3).

**⚠️ Finding (chưa chạy thực tế được):**
- `agent.build.dockerfile: 05-scaling-reliability/advanced/Dockerfile` (`docker-compose.yml:21`) — file/folder `advanced/` **không tồn tại** trong repo (chỉ có `develop/` và `production/`, và không có `Dockerfile` nào trong `05-scaling-reliability/`).
- `05-scaling-reliability/production/` cũng **chưa có `requirements.txt`** (cần `fastapi`, `uvicorn[standard]`, `redis`, `pydantic`) để build image.
- `env_file: .env.local` (`docker-compose.yml:27`) cũng chưa tồn tại.

→ Compose file này hiện chưa thể `docker compose up --scale agent=3` được ngay — phần phân tích trên dựa trên đọc code/cấu hình, chưa test runtime thực tế (khác với Part 2, nơi mọi thứ đã build & run thành công).

### Exercise 5.5: Test stateless (`test_stateless.py`)

- Script gửi **5 câu hỏi liên tiếp** tới `http://localhost:8080/chat` với cùng `session_id` (lần đầu `session_id=None` → server tự tạo).
- In ra `served_by` của từng response, gom vào `instances_seen` (set) — nếu `len(instances_seen) > 1` → in `"✅ All requests served despite different instances!"`, chứng minh load balancing hoạt động xuyên suốt.
- Cuối cùng `GET /chat/{session_id}/history` — kiểm tra **toàn bộ 10 messages** (5 cặp user/assistant) đều có mặt, dù được nhiều instance khác nhau xử lý → chứng minh state nằm ở Redis (chung), không phụ thuộc instance nào trả lời request cuối.
- **Chưa chạy thực tế** — phụ thuộc vào việc fix finding ở Exercise 5.4 (thiếu `Dockerfile`, `requirements.txt`, `.env.local` cho `05-scaling-reliability/production/`).
