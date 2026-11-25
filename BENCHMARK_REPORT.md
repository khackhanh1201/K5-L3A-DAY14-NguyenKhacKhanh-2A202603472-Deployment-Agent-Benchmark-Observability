# Benchmark & Observability Report — Day 14

> Copy file này thành `BENCHMARK_REPORT.md` trong repo của bạn và điền đầy đủ.
> Mọi con số phải lấy từ report JSON / CSV / screenshot thật, không ước lượng.

**Họ tên:** Nguyễn Khắc Khánh
**Ngày chạy:** 2026-11-25
**Môi trường:** Windows 11 + WSL2 Ubuntu (Python 3.11, Docker Desktop 29.9.3)

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | Đảm bảo 95% người dùng nhận phản hồi dưới 0.5 giây với mock LLM (~150–220ms), tránh trải nghiệm chờ đợi lâu |
| Error rate `/ask` | < 2.0 % | Cho phép sai số nhỏ từ lỗi upstream tạm thời (~1% mặc định), nhưng chặn ngay nếu lỗi tăng bất thường |
| Accuracy trên golden dataset | ≥ 80 % | Đảm bảo agent trả lời đúng đa số câu hỏi cốt lõi và không bị regression khi đổi prompt/retrieval |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.31 | 189.0 | 216.9 | 1706.3 | 1.33% | PASS |
| 10 | 36.27 | 186.2 | 222.0 | 1709.7 | 1.33% | PASS |
| 50 | 115.44 | 202.9 | 324.1 | 1705.5 | 1.33% | PASS |
| 100 | 71.45 | 340.9 | 3053.5 | 3261.3 | 1.33% | FAIL |

**Nhận xét:** RPS có tăng tuyến tính theo concurrency không? Vì sao?

Không tăng tuyến tính mãi: từ `c=1` lên `c=10` và `c=50`, RPS tăng mạnh (`4.31 → 36.27 → 115.44 req/s`) nhờ xử lý bất đồng bộ (`async/await`) trong khi latency p95 vẫn dưới `325ms`. Tuy nhiên đến `c=100`, hệ thống chạm điểm bão hòa (saturation) của 1 process uvicorn + tài nguyên CPU/event-loop, khiến request phải xếp hàng: p95 tăng vọt lên `3053.5ms` (vi phạm SLO 500ms) và throughput tụt xuống `71.45 req/s`.

### 2.2 Tail latency

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 189.5 | 216.9 | 223.2 |
| 0.03 (mặc định) | 244.8 | 221.0 | 1707.0 |
| 0.1 | 324.2 | 1682.8 | 1722.4 |

**Nhận xét:** Chỉ số nào nhạy nhất với request chậm? Vì sao SLO không nên đặt theo average?

`p99` và `p95` nhạy nhất với request chậm: khi `MOCK_SLOW_RATE` tăng từ `0` lên `0.03`, `avg` chỉ tăng nhẹ (`189.5ms → 244.8ms`) nhưng `p99` nhảy vọt hơn 7.6 lần (`223.2ms → 1707.0ms`); khi lên `0.1`, `p95` cũng nhảy vọt lên `1682.8ms`. Giá trị trung bình (average) bị số đông request nhanh làm phẳng nên che giấu trải nghiệm tệ của nhóm người dùng gặp tail latency; do đó SLO phải đặt theo `p95`/`p99`.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 57.4 | 109.3 | 113.7 |
| Tổng cost (USD) | $0.000408 | $0.000764 | $0.000790 |
| Cost / câu đúng (USD) | $0.00040815 | $0.00003639 | $0.00003436 |
| Quality gate | FAIL | PASS | PASS |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm entry từ khóa `["jwt"]` giải thích JWT so với API key vào `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm entry từ khóa `["hpa", "kubernetes", "autoscal"]` giải thích Kubernetes HPA vào `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` |

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**

`v2` (có bước retrieval RAG) tốn gần gấp đôi token mỗi câu (`109.3–113.7` so với `57.4`) và thêm ~20ms latency retrieval, nhưng accuracy tăng vọt từ `4.3%` lên `91.3%` (và `100.0%` sau khi bổ sung KB). Nhờ vậy, **cost / câu đúng** của `v2` (`$0.00003436`) rẻ hơn hơn **11 lần** so với `v1` (`$0.00040815`). Ngoài ra, đã thêm workflow CI [`.github/workflows/eval.yml`](.github/workflows/eval.yml) chạy `eval/run_eval.py --min-accuracy 0.85` (trả exit code `1` khi fail để chặn merge PR).

---

## 4. Observability stack (production)

### 4.1 Screenshot

- [ ] Dashboard Grafana khi đang có tải (thấy rõ Golden Signals)
- [ ] Một trace chậm trong Jaeger (thấy span nào chiếm thời gian)
- [ ] Một trace lỗi trong Jaeger (có event `exception`)
- [ ] Trang Alerts của Prometheus với ít nhất 1 alert đang **firing**

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | | | |
| B | `MOCK_SLOW_RATE=0.3` | | | |
| C | `LLM_MODEL=mock-llm-large` | | | |
| D | `docker pause` 1 instance | | | |
| E | `AGENT_VERSION=v1` | | | |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

...

### 4.3 Debug bằng trace

Chọn 1 trace chậm nhất bạn tìm được:

- `trace_id`: ...
- Tổng thời gian: ... ms
- Span chiếm nhiều thời gian nhất: ... (... ms, ...%)
- Dòng log tương ứng (grep theo `trace_id`):

```json

```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | | | | | |
| 3 | | | | | |

**Giải thích kết quả:** Scale có cải thiện p95 không? Nút thắt nằm ở đâu?

...

---

## 6. Kết luận

- 3 điều bạn học được về benchmark & observability:
  1. ...
  2. ...
  3. ...
- Nếu đưa agent này lên production thật, bạn sẽ thêm metric / alert nào nữa?

...
