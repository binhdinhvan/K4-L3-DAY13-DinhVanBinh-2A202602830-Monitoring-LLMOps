# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Đinh Văn Bình
- **MSSV:** 2A202602830
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/binhdinhvan/K4-L3-DAY13-DinhVanBinh-2A202602830-Monitoring-LLMOps
- **Commit SHA cuối:**
- **Challenge ID:** `day13-k4-l3b-monitoring-llmops-v1`
- **Tên project Langfuse cá nhân:** `day13-k4-l3b-2A202602830`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 30/100 | 100/100 | Đạt điểm tuyệt đối, đầy đủ correlation_id, enrichment và PII scrubbing |
| `validate_dashboard.py` | 6/6 panel | 6/6 panel | Hợp lệ 6/6 panel theo dashboard contract |
| `pytest` | 22 passed | 22 passed | Toàn bộ 22/22 unit tests đều pass |
| Số traces hợp lệ | 10 | ~90 traces | Traces phân cấp root -> retrieval -> generation đầy đủ |
| Số PII leak | 0 | 0 | Không có rò rỉ PII thô |
| Latency P95 / TTFT P95 | 1600ms / 50ms | 580ms / 50ms | Độ trễ P95 cải thiện sau khi tối ưu |
| Retrieval success rate | 100% | 100% | 100% lượt truy xuất tài liệu thành công |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** 
  - Tại `CorrelationIdMiddleware` (`app/middleware.py`), trước mỗi request ta gọi `clear_contextvars()` để xóa context cũ tránh rò rỉ dữ liệu giữa các request.
  - Trích xuất header `x-request-id` từ request client gửi lên; nếu không có, middleware tự sinh mới theo định dạng `req-<8-char-hex>` bằng `f"req-{uuid.uuid4().hex[:8]}"`.
  - Gắn ID này vào `structlog.contextvars` bằng `bind_contextvars(correlation_id=correlation_id)` và lưu vào `request.state.correlation_id`.
  - Cuối chu trình request, middleware tính toán thời gian xử lý và gắn lại vào response header: `x-request-id` và `x-response-time-ms`.

- **Các metadata được ghi vào structured log:**
  - Nhóm định danh request & bối cảnh: `correlation_id`, `ts` (ISO UTC), `level`, `service`, `event`, `env`.
  - Nhóm người dùng & tính năng: `user_id_hash` (băm SHA-256 rút gọn 12 ký tự), `session_id`, `feature`, `model`.
  - Nhóm đo lường hiệu năng & chi phí (ở `response_sent`): `latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success`.
  - Nhóm nội dung an toàn: `payload` chứa `message_preview` và `answer_preview` đã được rút gọn và scrub PII.

- **Cách bảo đảm PII được scrub trước khi ghi:**
  - Định nghĩa danh mục mẫu regex `PII_PATTERNS` trong `app/pii.py` cho email, số điện thoại Việt Nam (+84 hoặc 0, có khoảng trắng/chấm/gạch nối), số CCCD 12 chữ số, số thẻ tín dụng, số hộ chiếu.
  - Viết processor `scrub_event` trong `app/logging_config.py` duyệt đệ quy toàn bộ giá trị chuỗi (cả nested dict/list) trong `event_dict` và thay thế các chuỗi nhạy cảm bằng tag `[REDACTED_<TYPE>]`.
  - Đăng ký `scrub_event` vào pipeline của `structlog.configure(processors=[...])` trước bước render JSON và ghi file (`JsonlFileProcessor`). Nhờ đó, log trước khi xuất ra đĩa hay stdout đều đã được làm sạch hoàn toàn.

- **Cách kiểm chứng kết quả:**
  - Chạy test suite `python -m pytest -q` đảm bảo toàn bộ 22/22 unit tests pass.
  - Reset file `data/logs.jsonl` và chạy workload mẫu `python scripts/load_test.py`.
  - Chạy bộ kiểm tra tự động `python scripts/validate_logs.py` đạt điểm tuyệt đối **100/100**, xác nhận: đủ correlation_id, đủ metadata enrichment, không rò rỉ PII và định dạng JSON hợp lệ.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:**
  - Cấu hình API key riêng vào `.env` với public key `pk-lf-95048998-66c0-4503-a68d-798166539725` trỏ vào project cá nhân `day13-k4-l3b-2A202602830` trên Langfuse Cloud.
  - Trên giao diện Langfuse (ảnh `06-trace-list.png`), tiêu đề project hiển thị đúng tên cá nhân `day13-k4-l3b-2A202602830`.
  - Mọi trace đều mang timestamp thực tế khi chạy load test trên máy local và khớp chính xác các mã `correlation_id` sinh ra trong `data/logs.jsonl`.

- **Cấu trúc root/retrieval/generation observations:**
  - Root observation: `day13-agent-request` (trace root).
  - Lớp 1: `lab-agent-run` (agent span bao trùm toàn bộ chu trình xử lý của agent).
  - Lớp 2 (Child observations):
    - `retrieval` (loại `retriever`): đo đạc bước truy xuất tri thức/ngữ cảnh, lưu `query_preview` và `doc_count`.
    - `generation` (loại `generation`): đo đạc cuộc gọi LLM sinh câu trả lời, ghi nhận model (`claude-sonnet-4-5`), `prompt`, `usage_details` (`tokens_in`, `tokens_out`), `cost_usd` và `ttft_ms`.

- **Cách nối trace với log:**
  - Tại mỗi request, `CorrelationIdMiddleware` sinh hoặc nhận `correlation_id`.
  - Trong log, `correlation_id` được gắn vào tất cả các log events (`request_received`, `response_sent`, `request_failed`).
  - Trong trace, `correlation_id` được truyền vào `propagate_attributes(metadata={"correlation_id": correlation_id})` và được gắn vào span metadata của root và child span.
  - Khi điều tra sự cố, ta chỉ cần copy `correlation_id` từ log đem tìm kiếm trực tiếp trên Langfuse để mở ra đúng trace tương ứng.

- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version `1` (gắn nhãn `baseline` và `production`)
- **Version/label candidate:** Version `2` (gắn nhãn `candidate`)
- **Trace ID của mỗi version:**
  - Trace ID dùng Version 1: `c8e35e20dea09096f370f5621465dc81` (`prompt_version: 1`, `prompt_label: production`, `correlation_id: req-6fece989`)
  - Trace ID dùng Version 2: Trace sinh ra khi promote Version 2 lên `production` với request `req-e93930e0` (`prompt_version: 2`)
- **Cách promote và rollback `production`:**
  - **Promote:** Trên giao diện Langfuse Prompts, chọn Version 2 (`candidate`), mở popover quản lý nhãn và tích chọn `production` rồi lưu lại. Lúc này nhãn `production` tự động chuyển từ Version 1 sang Version 2 mà không cần sửa code.
  - **Rollback:** Khi cần quay lại bản ổn định trước đó, chọn lại Version 1, tích chọn lại `production` và lưu lại. Hệ thống API lập tức kéo và phục vụ Version 1 cho các request tiếp theo. Evidence được lưu tại `evidence/10-prompt-rollback.png`.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:**
  - `latency`: Đo percentiles P50, P95, P99 của latency và TTFT P95 từ event `response_sent`. Ngưỡng cảnh báo P95 <= 3000ms.
  - `traffic`: Đo số lượng request và tốc độ request theo phút (`rate_per_minute`) từ event `request_received`. Ngưỡng >= 1 req/min.
  - `errors`: Đo tỷ lệ lỗi `error_rate_pct` (request_failed / request_received), đếm theo `error_type` và đo tỷ lệ thành công của retrieval `tool_success_rate_pct`. Ngưỡng error rate <= 2%.
  - `cost`: Đo tổng chi phí USD theo phút và tổng lũy kế. Ngưỡng tổng chi phí <= $2.5 USD.
  - `tokens`: Đo tổng input token (`tokens_in`) và output token (`tokens_out`). Ngưỡng tổng token <= 50,000 tokens.
  - `quality`: Đo điểm chất lượng trung bình (`quality_score`) từ 0 đến 1. Ngưỡng trung bình >= 0.75.
  - Cấu hình contract được lưu tại `config/dashboard.yaml` và đạt kiểm định `6/6 panel` với `validate_dashboard.py`.

- **SLO và lý do chọn:**
  - Primary SLO: `fast_successful_requests` với mục tiêu 99.5% request trong chu kỳ 28 ngày đạt chuẩn: phải thành công (có event `response_sent`) và có `latency_ms <= 3000ms`.
  - Lý do chọn: Với một AI API tương tác người dùng, 3000ms là ngưỡng thời gian phản hồi chấp nhận được để không làm gián đoạn trải nghiệm người dùng, và mục tiêu 99.5% đảm bảo tính sẵn sàng cao mà vẫn chừa không gian đổi mới (error budget) cho hệ thống.

- **Cách tính error budget:**
  - Với SLO = 99.5% trong cửa sổ 28 ngày, Error Budget tương ứng là `100% - 99.5% = 0.5%`.
  - Ý nghĩa định lượng: Nếu hệ thống nhận 10,000 request trong 28 ngày, Error Budget cho phép tối đa `10,000 * 0.5% = 50 request` không đạt (bị lỗi HTTP 500 hoặc thời gian phản hồi > 3000ms). Khi số request vi phạm vượt quá 50, error budget bị cạn kiệt, cảnh báo đội ngũ kỹ thuật cần tạm ngừng phát hành tính năng mới để tập trung tối ưu hóa độ ổn định.

- **Ba alert và runbook tương ứng:**
  - Alert 1: `HighLatencyP95` (Warning, `p95(latency_ms) > 3000ms` duy trì 5 phút). Runbook: `docs/alerts.md#alert-1`.
  - Alert 2: `HighErrorRate` (Critical, `error_rate_pct > 2%` duy trì 3 phút). Runbook: `docs/alerts.md#alert-2`.
  - Alert 3: `LowRetrievalSuccessRate` (Warning, `retrieval_success_rate_pct < 90%` duy trì 5 phút). Runbook: `docs/alerts.md#alert-3`.

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3b-monitoring-llmops-v1`
- **Khoảng thời gian điều tra:** `2026-09-30T04:26:47Z – 2026-09-30T04:27:01Z` (UTC)
- **Triệu chứng từ metrics:** Metric cho thấy **latency** bất thường trong khoảng thời gian điều tra — tất cả 5 challenge request của feature `monitoring` có `latency_ms ≈ 2652–2653ms`, vượt ngưỡng bình thường (~160ms) gấp ~16 lần. Load test ghi nhận end-to-end latency lên tới **7973ms – 13287ms** (bao gồm cả thời gian xếp hàng chờ). Incident type được xác nhận: `rag_slow`.
- **Log line và correlation ID liên quan:** `event=response_sent`, `correlation_id=req-90313e79`, `feature=monitoring`, `latency_ms=2652`, `tool_name=retrieval`, `tool_success=true`, `ts=2026-09-30T04:26:50.427015Z`. (Xem `evidence/13-incident-log.png`)
- **Trace ID và span gây ảnh hưởng:** Tìm kiếm `correlation_id=req-90313e79` trên Langfuse → Trace tree cho thấy span **`retrieval`** có duration bất thường cao trong khi span `generation` bình thường. Trace ID xem tại `evidence/14-incident-trace.png`.
- **Root cause:** Span `retrieval` (as_type="retriever") bị làm chậm nhân tạo bởi incident injector kích hoạt flag `rag_slow=True`, mô phỏng việc vector store hoặc embedding lookup bị quá tải, khiến bước truy xuất tài liệu mất thêm ~2500ms so với bình thường.
- **Fix action:** Gọi endpoint `POST /admin/reset-incidents` (hoặc `inject_incident.py` với tham số reset) để tắt flag `rag_slow`. Xác nhận bằng cách chạy lại load test và kiểm tra `latency_ms` trở về dưới 200ms. Nếu trong production: scale out vector store, tăng connection pool, hoặc thêm caching tầng retrieval.
- **Preventive measure:** Alert `HighLatencyP95` (`p95(latency_ms) > 3000ms` kéo dài 5 phút) trong `config/alert_rules.yaml` sẽ kích hoạt cảnh báo sớm. Runbook `docs/alerts.md#alert-1` hướng dẫn kỹ sư kiểm tra span `retrieval` trên Langfuse và xác định nguyên nhân chậm. Ngoài ra, cần thêm circuit breaker cho vector store và giám sát độ trễ của từng span riêng biệt.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Sử dụng `@observe(as_type="retriever")` và `@observe(as_type="generation")` cho các method con trong agent thay vì một span đơn duy nhất. Quyết định này giúp Langfuse hiển thị rõ waterfall phân cấp `lab-agent-run → retrieval → generation`, từ đó trong bài toán `rag_slow`, ta ngay lập tức xác định được span `retrieval` là nguyên nhân — không phải `generation` — chỉ bằng cách nhìn vào duration của từng span.
- **Một lỗi/blocker đã gặp:** Ban đầu, metadata `correlation_id` không xuất hiện trong trace Langfuse vì gọi `propagate_attributes()` sai thời điểm (sau khi span đã đóng). Phải debug bằng cách so sánh log `correlation_id` với trace metadata trên Langfuse UI.
- **Cách tìm nguyên nhân và xử lý:** Đọc tài liệu Langfuse SDK v4 về lifecycle của `@observe` decorator và phát hiện ra `propagate_attributes()` phải được gọi bên trong hàm được decorate, trước khi return. Sau khi sửa đúng thứ tự, metadata xuất hiện đầy đủ trong trace.
- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics (dashboard latency) cho tín hiệu có vấn đề và khoanh vùng thời gian. Logs (structured JSON với `correlation_id`) giúp tìm chính xác request nào bị ảnh hưởng và xác nhận triệu chứng (`latency_ms`, `tool_name`). Traces (Langfuse) cho thấy toàn bộ cây span bên trong request, chỉ ra chính xác bước nào (`retrieval` vs `generation`) gây ra độ trễ — ba tầng này phối hợp với nhau như một phễu thu hẹp dần từ "hệ thống" xuống "dòng code".
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Prompt versioning cho phép thử nghiệm cải tiến prompt an toàn (version `candidate`) mà không ảnh hưởng production. Nếu version mới tăng token_out → tăng cost_usd → cảnh báo `cost` panel, ta rollback lại version cũ trong vòng vài giây. SLO định lượng hóa kỳ vọng chất lượng dịch vụ; khi error budget sắp cạn, nhóm ưu tiên ổn định hơn tính năng mới.
- **Điều quan trọng nhất đã học:** Ba tín hiệu observability (metrics, logs, traces) phải được thiết kế để liên kết với nhau từ đầu thông qua `correlation_id`. Nếu không có điểm neo chung này, việc điều tra sự cố trong production sẽ mất rất nhiều thời gian phỏng đoán.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Alert rules trong `config/alert_rules.yaml` là thiết kế tĩnh, chưa được kết nối với hệ thống alerting thực tế (ví dụ: PagerDuty, Slack webhook). Trong production, cần tích hợp thêm bước này để alerts thực sự kích hoạt được oncall.

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
