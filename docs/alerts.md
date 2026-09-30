# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert mẫu để tham khảo

Ví dụ dưới đây minh họa mức độ cụ thể cần có. Học viên không cần copy nguyên, nhưng ba alert trong bài nộp nên rõ ràng tương tự: điều kiện là gì, kéo dài bao lâu, ảnh hưởng tới user ra sao và người trực cần kiểm tra gì trước.

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` trong 5 phút
- Ảnh hưởng tới người dùng: người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh các span chính để xác định bước nào bất thường.
- Mitigation tạm thời: dựa trên evidence thực tế để rollback prompt, khôi phục cấu hình liên quan, tắt practice scenario hoặc giảm tải khi demo.
- Owner: `student-<MSSV>`

## Alert 1

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: `p95(latency_ms)` trong bảng `latency` của dashboard
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` duy trì trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng phải chờ lâu trước khi nhận được phản hồi từ AI API, trải nghiệm giảm sút
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel `latency` để xác nhận xu hướng P95/P99 và mốc thời gian bắt đầu tăng đột biến.
  2. Lọc file `data/logs.jsonl` trong khung thời gian đó, lấy một `correlation_id` của request có `latency_ms > 3000`.
  3. Mở trace có cùng `correlation_id` trên Langfuse, so sánh thời gian của span `retrieval` và span `generation` để xác định bước nào là thủ phạm gây nghẽn.
- Mitigation tạm thời:
  - Nếu span `generation` chậm do prompt mới làm output quá dài: rollback prompt `production` về version trước đó trên Langfuse.
  - Nếu span `retrieval` chậm do vector store/network: bật cache tài liệu hoặc chuyển sang fallback local docs.
- Owner: `student-2A202602830`

## Alert 2

- Tên: `HighErrorRate`
- Severity: `critical`
- Duration: `3m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: `error_rate_pct` trong bảng `errors` của dashboard và guardrail `error_rate_pct_max: 2`
- Điều kiện và thời gian duy trì: `error_rate_pct > 2%` duy trì trong 3 phút
- Ảnh hưởng tới người dùng: Người dùng nhận mã lỗi HTTP 500 hoặc thông báo lỗi hệ thống, không nhận được câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel `errors` để xem tỷ lệ lỗi hiện tại và phân bố theo `error_type`.
  2. Lọc log sự kiện `request_failed` trong `data/logs.jsonl`, trích xuất `correlation_id` và `error_type` (ví dụ `RuntimeError`, `RateLimitError`).
  3. Mở trace trên Langfuse theo `correlation_id` để kiểm tra stack trace lỗi chi tiết và span bị fail.
- Mitigation tạm thời:
  - Kiểm tra kết nối dịch vụ ngoài (LLM backend / Vector database).
  - Khôi phục cấu hình hoặc rollback phiên bản prompt / code vừa triển khai.
  - Nếu do bị quá tải hoặc DDoS, áp dụng rate limiting tạm thời ở middleware.
- Owner: `student-2A202602830`

## Alert 3

- Tên: `LowRetrievalSuccessRate`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: `retrieval_success_rate_pct` trong bảng `errors` và guardrail `retrieval_success_rate_pct_min: 90`
- Điều kiện và thời gian duy trì: `retrieval_success_rate_pct < 90%` duy trì trong 5 phút
- Ảnh hưởng tới người dùng: Mô hình không truy xuất được tài liệu phù hợp (RAG context rỗng hoặc fail), dẫn đến câu trả lời thiếu chính xác hoặc ảo giác
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel `errors`, kiểm tra chỉ số `tool_success_rate_pct` cho công cụ `retrieval`.
  2. Lọc log các request có `tool_name == "retrieval"` và `tool_success == false` hoặc `payload.doc_count == 0`.
  3. Xem trace tương ứng trên Langfuse, kiểm tra input query tìm kiếm và metadata `prompt_fetch_error` / `doc_count`.
- Mitigation tạm thời:
  - Khởi động lại dịch vụ retrieval/vector database.
  - Kiểm tra trạng thái index của kho tài liệu kiến thức.
  - Bật cơ chế fallback dùng knowledge base cục bộ nếu remote retriever bị treo.
- Owner: `student-2A202602830`
