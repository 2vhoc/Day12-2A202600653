# Lab 06 Complete - Production AI Agent

Đây là bài nộp Lab 6 của Day 12. Project này đóng gói một AI agent dạng API service, có Docker, health check, readiness check, API key authentication, rate limiting, cost guard, structured logging và cấu hình theo environment variables.

Agent đang dùng `utils/mock_llm.py`, nên không cần OpenAI API key thật để chạy và kiểm thử deployment.

## Tính Năng

- FastAPI service với endpoint `/ask`
- API key authentication qua header `X-API-Key`
- Rate limiting theo API key
- Cost guard theo daily budget
- Health check: `GET /health`
- Readiness check: `GET /ready`
- Metrics endpoint: `GET /metrics`
- Structured JSON logging
- Graceful shutdown với `SIGTERM`
- Multi-stage Dockerfile
- Docker Compose stack gồm `agent` và `redis`
- Không hardcode production secrets trong source code

## Cấu Trúc

```text
06-lab-complete/
├── app/
│   ├── main.py              # FastAPI app: auth, rate limit, cost guard, endpoints
│   └── config.py            # 12-factor config từ environment variables
├── utils/
│   └── mock_llm.py          # Mock LLM để chạy offline
├── Dockerfile               # Multi-stage production image
├── docker-compose.yml       # Agent + Redis local stack
├── requirements.txt         # Python dependencies
├── .env.example             # Environment template
├── .dockerignore            # Docker build ignore
├── railway.toml             # Railway deploy config
├── render.yaml              # Render deploy config
├── check_production_ready.py # Production readiness checker
└── README.md
```

## Cấu Hình

Tạo file env local từ template:

```bash
cp .env.example .env.local
```

Các biến quan trọng:

```text
HOST=0.0.0.0
PORT=8000
ENVIRONMENT=development
DEBUG=false
AGENT_API_KEY=dev-key-change-me-in-production
JWT_SECRET=dev-jwt-secret-change-in-production
RATE_LIMIT_PER_MINUTE=20
DAILY_BUDGET_USD=5.0
REDIS_URL=redis://redis:6379/0
```

Khi deploy production, cần đổi:

```text
ENVIRONMENT=production
DEBUG=false
AGENT_API_KEY=<secret-key>
JWT_SECRET=<secret-value>
```

## Chạy Local Với Docker Compose

Nếu Docker cần quyền sudo:

```bash
sudo docker compose up --build
```

Nếu user đã có quyền Docker:

```bash
docker compose up --build
```

Service mặc định chạy ở:

```text
http://localhost:8000
```

## Test API

Health check:

```bash
curl http://localhost:8000/health
```

Readiness check:

```bash
curl http://localhost:8000/ready
```

Không có API key sẽ bị từ chối:

```bash
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```

Kết quả mong đợi:

```text
HTTP/1.1 401 Unauthorized
```

Gọi API có key:

```bash
API_KEY=$(grep AGENT_API_KEY .env.local | cut -d= -f2)

curl -X POST http://localhost:8000/ask \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is deployment?"}'
```

Test rate limiting:

```bash
API_KEY=$(grep AGENT_API_KEY .env.local | cut -d= -f2)

for i in $(seq 1 25); do
  curl -sS -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost:8000/ask \
    -H "X-API-Key: $API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"question\":\"rate limit test $i\"}"
done
```

Sau khi vượt `RATE_LIMIT_PER_MINUTE`, service sẽ trả `429`.

## Build Docker Image

```bash
sudo docker build -t lab06-production-agent .
```

Run image:

```bash
sudo docker run --rm -p 8000:8000 \
  -e ENVIRONMENT=development \
  -e AGENT_API_KEY=dev-key-change-me-in-production \
  -e JWT_SECRET=dev-jwt-secret-change-in-production \
  lab06-production-agent
```

## Deploy

Project có sẵn cấu hình Railway và Render:

- `railway.toml`
- `render.yaml`

Khi deploy, set các environment variables production trong dashboard của platform:

```text
ENVIRONMENT=production
DEBUG=false
AGENT_API_KEY=<secret-key>
JWT_SECRET=<secret-value>
RATE_LIMIT_PER_MINUTE=10
DAILY_BUDGET_USD=10.0
```

## Production Readiness Check

Chạy:

```bash
python check_production_ready.py
```

Log kiểm tra hiện tại:

```text
=======================================================
  Production Readiness Check — Day 12 Lab
=======================================================

📁 Required Files
  ✅ Dockerfile exists
  ✅ docker-compose.yml exists
  ✅ .dockerignore exists
  ✅ .env.example exists
  ✅ requirements.txt exists
  ✅ railway.toml or render.yaml exists

🔒 Security
  ✅ .env in .gitignore
  ✅ No hardcoded secrets in code

🌐 API Endpoints (code check)
  ✅ /health endpoint defined
  ✅ /ready endpoint defined
  ✅ Authentication implemented
  ✅ Rate limiting implemented
  ✅ Graceful shutdown (SIGTERM)
  ✅ Structured logging (JSON)

🐳 Docker
  ✅ Multi-stage build
  ✅ Non-root user
  ✅ HEALTHCHECK instruction
  ✅ Slim base image
  ✅ .dockerignore covers .env
  ✅ .dockerignore covers __pycache__

=======================================================
  Result: 20/20 checks passed (100%)
  🎉 PRODUCTION READY! Deploy nào!
=======================================================
```

## Ghi Chú

- File `.env.local` chỉ dùng local, không commit lên Git.
- `OPENAI_API_KEY` có thể để trống vì project dùng mock LLM.
- Nếu `/health` bị lỗi `MutableHeaders has no attribute pop`, kiểm tra `app/main.py` phải dùng:

```python
if "server" in response.headers:
    del response.headers["server"]
```
