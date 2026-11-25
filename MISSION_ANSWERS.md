# Day 14 Lab — Mission Answers

> **Lưu ý:** Các phần số đo thực tế (`docker images`, output `curl`, `test_stateless.py`, `check_production_ready.py`, URL deploy) được đánh dấu `[TODO: ...]` để điền đúng output bạn tự chạy trong WSL2/cloud (theo [RULES.md](RULES.md), không bịa số đo).

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns phát hiện trong `01-localhost-vs-production/develop/app.py`
1. **Hardcode secret trong source code (dòng 17–18):** `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`. Nếu push lên GitHub sẽ lộ ngay thông tin nhạy cảm.
2. **Không quản lý cấu hình theo môi trường (dòng 21–22):** Hardcode `DEBUG = True`, `MAX_TOKENS = 500`, không đọc từ biến môi trường (vi phạm 12-Factor App).
3. **Dùng `print()` thay cho structured logging và in lộ secret (dòng 33–34):** `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` làm lộ API key ra log và khó parse/tìm kiếm trên hệ thống log tập trung.
4. **Thiếu health/readiness check endpoint:** Không có `/health` hay `/ready` để platform/load balancer biết service còn sống hay đã sẵn sàng nhận traffic.
5. **Hardcode host/port và bật `reload=True` (dòng 49–54):** `host="localhost"` chỉ nhận kết nối nội bộ (không nhận traffic từ ngoài container), `port=8000` không nhận biến `PORT` do cloud cấp, và `reload=True` gây tốn tài nguyên/rủi ro trong production.
6. **Không xử lý graceful shutdown:** Không có lifecycle hook (`lifespan`) hay xử lý `SIGTERM`, dễ cắt ngang request đang xử lý khi container restart/deploy.

### Exercise 1.2: Kết quả chạy bản `develop`
```text
POST /ask?question=Hello HTTP/1.1 -> 200 OK
Answer: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.
(Log console của develop/app.py còn in thẳng: [DEBUG] Using key: sk-hardcoded-fake-key-never-do-this)
```

### Exercise 1.3: So sánh `develop/app.py` vs `production/app.py`
| Feature | Develop (`develop/app.py`) | Production (`production/app.py`) | Tại sao quan trọng? |
|---|---|---|---|
| **Config** | Hardcode `OPENAI_API_KEY`, `DATABASE_URL`, `DEBUG`, `PORT=8000` | Đọc qua `settings` từ `.env` / biến môi trường (`config.py`) | Không lộ secret vào Git; cùng một artifact chạy được ở dev/staging/prod |
| **Health check** | Không có `/health`, `/ready` | Có `GET /health` (liveness) và `GET /ready` (readiness, trả 503 khi chưa sẵn sàng) | Platform biết khi nào cần restart container và load balancer biết khi nào được route traffic |
| **Logging** | `print("[DEBUG] ...")`, in cả API key | `logging` cấu hình JSON (`{"time": ..., "level": ..., "msg": ...}`), không log secret | Dễ thu thập, lọc và nối vết trên hệ thống log tập trung (Loki/Datadog) và an toàn bảo mật |
| **Shutdown** | Thoát đột ngột | Dùng `lifespan` (`is_ready = False` khi shutdown) và handler `SIGTERM` | Tránh làm rớt các request đang xử lý (in-flight) khi deploy phiên bản mới hoặc scale down |
| **Server binding** | `host="localhost"`, `port=8000`, `reload=True` | `host=settings.host` (`0.0.0.0`), `port=settings.port`, `reload=settings.debug` | `0.0.0.0` cần thiết để container nhận traffic từ bên ngoài; `PORT` lấy động từ cloud |

**Ví dụ dòng code `production/app.py` đã sửa so với `develop/app.py`:**
- `develop/app.py` dòng 34: `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` → `production/app.py` dòng 28–31 & 111–115 dùng `logging.basicConfig(format='{"time":"%(asctime)s","level":"%(levelname)s","msg":"%(message)s"}')` và `logger.info(json.dumps({"event": "agent_request", "question_length": len(question), ...}))` (chỉ log metadata, không log secret).
- `develop/app.py` dòng 51–53: `host="localhost", port=8000, reload=True` → `production/app.py` dòng 196–198: `host=settings.host, port=settings.port, reload=settings.debug`.

**Kết quả test bản `production/app.py`:**
```text
GET /health -> HTTP/1.1 200 OK
{"status":"ok","uptime_seconds":1040.8,"version":"1.0.0","environment":"development","timestamp":"2026-11-25T08:11:00.358981+00:00"}

GET /ready -> HTTP/1.1 200 OK
{"ready":true}

POST /ask -> HTTP/1.1 200 OK
{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","model":"gpt-4o-mini"}
```

---

## Part 2: Docker

### Exercise 2.1: Trả lời câu hỏi `02-docker/develop/Dockerfile`
1. **Base image:** `python:3.11` (full Debian + Python distribution).
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Tại sao `COPY requirements.txt` đứng trước `COPY ... app.py`?** Để tận dụng cơ chế Docker layer cache. Khi chỉ sửa code ứng dụng mà `requirements.txt` không đổi, Docker tái sử dụng lại layer `RUN pip install ...` đã cache thay vì cài lại toàn bộ thư viện.
4. **`CMD` vs `ENTRYPOINT` khác nhau thế nào?**
   - `CMD` định nghĩa lệnh/tham số mặc định khi chạy container và có thể bị ghi đè dễ dàng khi truyền command ở cuối `docker run`.
   - `ENTRYPOINT` cố định executable chính của container; các tham số truyền thêm ở `docker run` (hoặc từ `CMD`) sẽ được nối vào làm đối số cho `ENTRYPOINT`.

### Exercise 2.2: Kết quả gọi container `my-agent:develop`
```text
GET /health -> HTTP/1.1 200 OK
{"status":"ok","uptime_seconds":2.7,"container":true}

POST /ask?question=What%20is%20Docker%3F -> HTTP/1.1 200 OK
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: Multi-stage build & so sánh dung lượng
- **Stage 1 (`builder`):** Dùng `python:3.11-slim`, cài `gcc`, `libpq-dev` và chạy `pip install --no-cache-dir --user -r requirements.txt` vào `/root/.local`.
- **Stage 2 (`runtime`):** Dùng `python:3.11-slim` sạch, tạo non-root user `appuser`, chỉ `COPY --from=builder /root/.local /home/appuser/.local` và copy code (`main.py`, `utils/mock_llm.py`), thêm `HEALTHCHECK` và chạy bằng `USER appuser`.
- **Tại sao image nhỏ hơn?** Vì toàn bộ compiler (`gcc`), header (`libpq-dev`), cache apt/pip ở stage `builder` không được đưa sang image `runtime` cuối cùng.

**Kích thước đo được trên máy (`docker images | grep my-agent`):**
- `my-agent:develop`: `1.67 GB` (1670 MB, content 426 MB)
- `my-agent:advanced`: `240 MB` (content 57.4 MB)
- Chênh lệch: giảm **`85.6%`** (giảm ~1.43 GB)

### Exercise 2.4: Docker Compose stack (`02-docker/production/`)
**Sơ đồ kiến trúc các service:**
```text
Client (localhost:80)
   │
   ▼
[ nginx ] (ports: 80, 443; reverse proxy/LB)
   │
   ▼
[ agent ] (port 8000 internal, build từ multi-stage Dockerfile, depends_on redis & qdrant)
   ├──▶ [ redis ] (redis:7-alpine, session cache & rate limiting)
   └──▶ [ qdrant ] (qdrant/qdrant:v1.9.0, vector DB cho RAG)
```
Các service giao tiếp qua Docker network `internal` bằng tên service (`redis://redis:6379/0`, `http://qdrant:6333`), chỉ `nginx` mở port `80`/`443` ra ngoài.

**Kết quả test qua Nginx:**
- [TODO: Dán output `curl -i http://localhost/health` và `curl -i -X POST http://localhost/ask ...`]

---

## Part 3: Cloud Deployment

### Exercise 3.1 / 3.2: Deploy Agent Part 6 lên Cloud
- **Platform chính:** [TODO: Railway hoặc Render]
- **Public URL:** `[TODO: https://...]`
- **Screenshots:** `screenshots/dashboard.png`, `screenshots/running.png`, `screenshots/test.png`

### So sánh `railway.toml` vs `render.yaml`
| Tiêu chí | `railway.toml` | `render.yaml` |
|---|---|---|
| **Mục đích** | Cấu hình build & deploy cho 1 service trên Railway (TOML) | Blueprint Infrastructure-as-Code cho nhiều service (YAML) |
| **Khai báo hạ tầng phụ trợ (Redis)** | Không khai báo trực tiếp Redis trong `railway.toml` (thêm service Redis qua Railway dashboard/CLI rồi nối `REDIS_URL`) | Có thể khai báo luôn service web + Redis add-on trong cùng 1 file `render.yaml` |
| **Cách khai báo build & healthcheck** | `[build] builder = "DOCKERFILE"`, `[deploy] healthcheckPath = "/health"` | Dưới `services:`: `type: web`, `runtime: docker`, `healthCheckPath: /health` |
| **Quản lý biến môi trường/secret** | Thường đặt qua `railway variables set` hoặc Railway Dashboard | Khai báo ngay danh sách `envVars` trong YAML; secret dùng `generateValue: true` hoặc `sync: false` để không lộ trong Git |

### Exercise 3.3: Phân tích pipeline GCP Cloud Run (`03-cloud-deployment/production-cloud-run/`)
Pipeline CI/CD trong `cloudbuild.yaml` và `service.yaml` gồm 4 bước tuần tự khi push lên nhánh `main`:
1. **Test (`test`):** Dùng container `python:3.11-slim`, cài dependencies và chạy `pytest tests/ -v`; nếu test fail thì dừng pipeline.
2. **Build (`build`):** Dùng `gcr.io/cloud-builders/docker` với `--cache-from` để build Docker image gắn tag `$COMMIT_SHA` và `latest`.
3. **Push (`push`):** Đẩy các tag image lên Google Container Registry (`gcr.io/$PROJECT_ID/ai-agent`).
4. **Deploy (`deploy`):** Dùng `gcloud run deploy` triển khai image `$COMMIT_SHA` lên Cloud Run (`asia-southeast1`), cấu hình tài nguyên (512Mi RAM, 1 CPU), scaling (`min-instances=1` tránh cold start, `max-instances=10`), biến môi trường và inject secret an toàn từ GCP Secret Manager (tương ứng cấu hình Knative Service + liveness `/health` + startup probe `/ready` trong `service.yaml`).

---

## Part 4: API Security

### Exercise 4.1–4.3: Kết quả kiểm tra thực tế (`curl -i`, đã che token)
- **4.1 (Develop — thiếu API key → 401, đúng key → 200):**
```text
# Không có X-API-Key:
HTTP/1.1 401 Unauthorized
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

# Có X-API-Key: secret-key-123:
HTTP/1.1 200 OK
{"question":"Hello","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```
- **4.2 (Production — `POST /auth/token` với `student`/`demo123` rồi gọi `/ask`):**
```text
# Không có Bearer token:
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}

# Có Authorization: Bearer <REDACTED_TOKEN>:
HTTP/1.1 200 OK
x-content-type-options: nosniff
x-frame-options: DENY
{"question":"Explain JWT","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","usage":{"requests_remaining":9,"budget_remaining_usd":1.6e-05}}
```
- **4.3 (Rate limiting — vòng lặp vượt 10 req/phút → 429):**
  - *Thuật toán trong `04-api-gateway/production/rate_limiter.py`:* **Sliding Window** dùng `deque` lưu timestamps request trong cửa sổ `window_seconds = 60`.
  - *Limit:* `rate_limiter_user` giới hạn `10` request/phút; `rate_limiter_admin` giới hạn `100` request/phút (chọn limiter theo `user["role"]`).
```text
Req 1..9: HTTP 200
Req 10: HTTP 429
Req 11: HTTP 429

HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
retry-after: 59
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
```

### Exercise 4.4: Cách hoạt động của Cost Guard
- **Mục tiêu:** Bảo vệ ngân sách LLM, tránh hóa đơn tăng đột biến.
- **Cách triển khai (trong `04-api-gateway/production/cost_guard.py` và bản Redis ở Part 6):**
  1. **Trước khi gọi LLM (`check_budget`):** Đọc tổng chi phí đã dùng trong ngày/tháng (từ Redis key theo ngày `cost:daily:YYYY-MM-DD`). Nếu `>= budget` → chặn ngay bằng **`HTTP 402 Payment Required`** trước khi tốn thêm token.
  2. **Sau khi gọi LLM (`record_cost`):** Tính chi phí từ số `input_tokens` và `output_tokens` theo đơn giá model, cộng dồn nguyên tử bằng `INCRBYFLOAT` vào Redis kèm `EXPIRE` để tự động dọn/reset sang chu kỳ mới.
- **Vì sao cần cả Rate Limit và Cost Guard:** Rate limit kiểm soát **tốc độ** (request/phút, chống spam/DoS), còn Cost guard kiểm soát **tổng tiền** (vì chỉ vài prompt rất dài hoặc model đắt tiền vẫn có thể đốt hết ngân sách dù nằm dưới ngưỡng rate limit).

---

## Part 5: Scaling & Reliability

### Exercise 5.1–5.5: Ghi chú và kết quả kiểm tra
1. **Phân biệt Liveness (`/health`) và Readiness (`/ready`):**
   - `/health` kiểm tra **process còn sống không**: luôn trả `200` khi tiến trình app đang chạy (kể cả khi Redis tạm mất kết nối). Nếu `/health` fail, orchestrator/platform sẽ **restart container**.
   - `/ready` kiểm tra **app có sẵn sàng nhận traffic không** (ping Redis/DB): trả `503` khi dependency chưa sẵn sàng để load balancer **ngừng route traffic mới** vào instance đó mà không cần restart container.
2. **Graceful shutdown:**
   - Khi nhận `SIGTERM`, app chuyển sang trạng thái ngừng nhận request mới, chờ các request in-flight xử lý xong (tránh rớt request đang chạy), log shutdown rồi thoát với exit code `0`.
3. **Stateless design & Load balancing (Exercise 5.3–5.5):**
   - Trong `05-scaling-reliability/production/`, conversation history được lưu trong **Redis** (`session:{session_id}`) thay vì RAM của từng instance. Nhờ vậy khi chạy 3 instance sau Nginx (`--scale agent=3`), request 1 có thể vào `agent-1`, request 2 vào `agent-2` nhưng vẫn đọc đủ lịch sử hội thoại.
   - **Kết quả chạy `python test_stateless.py` (stack `k5-day14-scaling` với `--scale agent=3`):**
```text
Session ID: 3022f62d-2e73-4bc6-b230-83d54f4ccf36
Request 1: [instance-146374]
Request 2: [instance-47f387]
Request 3: [instance-c9a5b2]
Request 4: [instance-146374]
Request 5: [instance-47f387]
Total requests: 5
Instances used: {'instance-47f387', 'instance-146374', 'instance-c9a5b2'}
✅ All requests served despite different instances!
Total messages in history: 10
✅ Session history preserved across all instances via Redis!
```

---

## Part 6: Final Project (`06-lab-complete/`)

- Output phần `📋 Checklist Deliverable` khi chạy `python check_production_ready.py` (đầy đủ, trong WSL2 có Docker):
```text
================================================================
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 43/43 checks passed
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
================================================================
```

---

## AI log

| Thời điểm | Hỏi / Yêu cầu AI gì | Nhận được gì | Tự sửa và tự kiểm tra thế nào |
|---|---|---|---|
| 2026-11-25 | Hướng dẫn chi tiết quy trình làm bài lab Day 14 trên Windows + WSL2 và cài đặt môi trường | Hướng dẫn chuyển sang WSL2, bật Docker Desktop WSL Integration, tạo venv Python 3.11 bằng `uv` do Python 3.14 không build được `pydantic-core==2.23.2` | Đã bật Docker trong WSL2 (`hello-world` chạy thành công), tạo `~/.venv-day14` Python 3.11 và cài đặt xong dependencies |
| 2026-11-25 | Kiểm tra và hỗ trợ hoàn thiện code Part 6 (`06-lab-complete/`) và khung trả lời `MISSION_ANSWERS.md` | Hoàn thiện các file `app/config.py`, `redis_client.py`, `auth.py`, `rate_limiter.py`, `cost_guard.py`, `main.py`, `Dockerfile`, `docker-compose.yml`, `.dockerignore`, `.env.example`, `railway.toml`, `render.yaml` và bản nháp lý thuyết `MISSION_ANSWERS.md` | Chạy `check_production_ready.py --static-only` kiểm tra tĩnh, và cần tự chạy `python check_production_ready.py` đầy đủ trong WSL2, tự chạy Part 1–5 và điền số đo thật vào các mục `[TODO]` |
