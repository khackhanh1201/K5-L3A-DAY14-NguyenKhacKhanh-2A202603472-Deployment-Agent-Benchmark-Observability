# Deployment Information

## Public URL
https://k5-l3a-day14-nguyenkhackhanh-2a202603472.onrender.com

## Platform
Render (Web Service + Render Key Value Redis)

## Test Commands

### Health Check
```bash
curl https://k5-l3a-day14-nguyenkhackhanh-2a202603472.onrender.com/health
# Expected: {"status": "ok"}
```

### Readiness Check
```bash
curl https://k5-l3a-day14-nguyenkhackhanh-2a202603472.onrender.com/ready
# Expected: {"status": "ready"}
```

### API Test (without API key -> 401)
```bash
curl -i -X POST https://k5-l3a-day14-nguyenkhackhanh-2a202603472.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

### API Test (with authentication -> 200)
```bash
curl -X POST https://k5-l3a-day14-nguyenkhackhanh-2a202603472.onrender.com/ask \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```

## Environment Variables Set
- `PORT` (injected automatically by Render)
- `REDIS_URL` (Render Key Value internal URL)
- `AGENT_API_KEY` (generated secret on Render Dashboard)
- `ENVIRONMENT` (`production`)
- `RATE_LIMIT_PER_MINUTE` (`10`)
- `DAILY_BUDGET_USD` (`1.0`)

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
