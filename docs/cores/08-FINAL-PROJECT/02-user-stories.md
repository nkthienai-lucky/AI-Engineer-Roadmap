# User Stories — GenAI Knowledge Assistant

> **Mục tiêu:** Biến tầm nhìn sản phẩm thành 5 kịch bản người dùng cụ thể, mỗi kịch bản được phân tích kỹ: acceptance ngầm (những điều phải đúng mà người dùng không nói ra), API/endpoint gợi ý để triển khai, và cách test để biết story đã xong.
> Nguồn mở rộng từ: [08-FINAL-PROJECT](README.md)

---

## Nguyên tắc chung khi viết user story

Mỗi story dạng *"Là [vai trò], tôi [hành động] để [kết quả]"* chỉ là phần nổi của tảng băng. Phần chìm — acceptance ngầm — mới là thứ quyết định story "xong" hay "xong giả": ví dụ "upload PDF" nghe đơn giản, nhưng upload file 200 trang mất 3 phút thì xử lý thế nào? Upload file hỏng báo ra sao? Các câu hỏi đó phải được trả lời **trước khi code**, không phải sau khi bug xuất hiện.

## 1. Upload PDF → hỏi đáp có trích dẫn nguồn

> *Là người dùng,* tôi upload 1 bộ tài liệu PDF → hỏi câu hỏi → nhận câu trả lời kèm **trích dẫn nguồn** (trang/đoạn).

**Acceptance ngầm:**
- Upload được **nhiều** PDF thành 1 "bộ tài liệu" (collection), không phải từng file một rời rạc.
- Quá trình ingest (parse → chunk → embed) là **bất đồng bộ**: user không phải ngồi chờ; có trạng thái "đang xử lý / xong / lỗi".
- Citation trỏ về đúng **trang + đoạn** trong file gốc, và bấm vào được (mở ra đoạn đó) — citation chỉ là text vô nghĩa nếu user không kiểm chứng được.
- File hỏng/scan không có text → báo lỗi rõ ràng, không im lặng nuốt file.
- Câu hỏi không có trong tài liệu → trả lời "không tìm thấy thông tin", **không bịa**.

**API/endpoint gợi ý:**
- `POST /api/collections` — tạo bộ tài liệu; `POST /api/collections/{id}/documents` (multipart) — upload PDF, trả về `document_id` + trạng thái ingest.
- `GET /api/documents/{id}/status` — poll trạng thái ingest (pending/processing/done/failed).
- `POST /api/chat` — body gồm `collection_id`, `question`, `history`; response SSE stream kèm `citations: [{document_id, page, chunk_id, quote}]`.

**Cách test:**
- Upload 1 PDF 200 trang → hỏi 5 câu có sẵn đáp án đúng → kiểm tra citation có trỏ đúng trang (so tay với PDF).
- Hỏi 1 câu ngoài lề ("thủ đô nước Pháp?") → phải từ chối.
- Chạy bộ 30 câu test qua Ragas → faithfulness > 0.8 (xem [06-acceptance-and-risks.md](06-acceptance-and-risks.md)).

## 2. Ảnh hóa đơn → JSON có cấu trúc

> *Là người dùng,* tôi upload ảnh hóa đơn → hệ thống trả về JSON có cấu trúc (số tiền, ngày, nhà cung cấp...).

**Acceptance ngầm:**
- JSON theo **schema cố định** (ví dụ `{vendor, date, total, currency, items[]}`) — không phải JSON tùy hứng mỗi lần khác kiểu.
- Trường nào không đọc được thì để `null` kèm độ tự tin thấp, **không đoán bừa**.
- Ảnh mờ/nghiêng/vỡ → trả về lỗi hoặc cảnh báo, không trả JSON rác.
- Kết quả JSON hiển thị cả ảnh gốc cạnh bên để user đối chiếu nhanh.

**API/endpoint gợi ý:**
- `POST /api/vision/extract` — multipart ảnh + tham số `schema` (mặc định "invoice"); response: `{fields: {...}, raw_text, confidence}`.
- Tuỳ chọn: `POST /api/vision/extract?to_rag=true` để đưa caption/JSON vào RAG context (xem [03-architecture-and-flows.md](03-architecture-and-flows.md)).

**Cách test:**
- Chuẩn bị 10 ảnh hóa đơn/biểu mẫu thật (đa dạng độ rõ, ngôn ngữ, bố cục) → chạy batch → chấm tay: đúng ≥ 8/10 ở mức field-level (số tiền, ngày, nhà cung cấp phải đúng).
- Test ảnh không phải hóa đơn (ảnh mèo) → hệ thống phải nói "không phải hóa đơn" hoặc JSON rỗng, không bịa số.

## 3. Chat thường giữ ngữ cảnh hội thoại

> *Là người dùng,* tôi chat bình thường (không cần tài liệu) → model trả lời, giữ ngữ cảnh hội thoại.

**Acceptance ngầm:**
- Không có `collection_id` → chat thuần LLM, không gọi retrieval (tránh latency vô nghĩa).
- "Giữ ngữ cảnh" = nhớ được **vài lượt gần nhất** (ví dụ 4–6 lượt): hỏi "nó là gì?" sau khi nói về "RAG" thì model hiểu "nó" là RAG.
- Streaming từng token (SSE) — user thấy chữ xuất hiện dần, không chờ 5 giây rồi cả khối đổ ra.
- Lịch sử hội thoại lưu được, reload trang không mất.
- Giới hạn `max_tokens` hợp lý để không sinh lan man (gắn với rủi ro 524 — xem [06-acceptance-and-risks.md](06-acceptance-and-risks.md)).

**API/endpoint gợi ý:**
- `POST /api/chat` với `collection_id = null` → nhánh chat thường (cùng endpoint, ít route hơn ít bug hơn).
- `GET /api/conversations` / `GET /api/conversations/{id}` — lịch sử; conversation lưu dạng list `[{role, content}]` gửi kèm mỗi request (đơn giản, đủ cho 1 user).

**Cách test:**
- Kịch bản 3 lượt: "RAG là gì?" → "Nó cần những thành phần nào?" → "Thành phần nào khó nhất?" — kiểm tra model trả lời đúng "nó"/"thành phần" được tham chiếu.
- Đo latency token đầu < 3s trên CPU (xem mục Performance ở [06-acceptance-and-risks.md](06-acceptance-and-risks.md)).

## 4. Feedback "không đúng" để cải thiện

> *Là người dùng,* tôi thấy câu trả lời không đúng → bấm "không đúng" → hệ thống ghi nhận (dữ liệu để cải thiện).

**Acceptance ngầm:**
- Nút feedback **không làm gián đoạn luồng chat** — bấm xong vẫn chat tiếp bình thường.
- Ghi nhận đủ ngữ cảnh: câu hỏi, câu trả lời, citations lúc đó, collection — để sau này dùng làm dữ liệu fine-tune/eval (mục 8 trong file nguồn).
- Feedback không cần lý do chi tiết (nút 1 click); nếu user muốn thì ô text tự do — bắt buộc nhập lý do sẽ khiến không ai bấm.
- Đếm được tổng feedback xấu/tốt (phục vụ story 5).

**API/endpoint gợi ý:**
- `POST /api/feedback` — body `{message_id, rating: "bad"|"good", comment?, snapshot: {question, answer, citations}}`; lưu bảng `feedback`.
- `GET /api/admin/feedback` — operator xem danh sách (ghép với story 5).

**Cách test:**
- Bấm "không đúng" → kiểm tra DB có record đầy đủ snapshot; chat tiếp bình thường.
- Từ snapshot ghi được, dựng được bộ câu hỏi eval mới (regression set) cho Ragas.

## 5. Operator xem log chi phí & hiệu năng

> *Là người vận hành,* tôi xem được log: bao nhiêu request, latency, chi phí token ước tính.

**Acceptance ngầm:**
- "Chi phí token ước tính" = số token × đơn giá tham chiếu (dù chạy local miễn phí, vẫn quy ra tiền để biết nếu chuyển sang API thì tốn bao nhiêu).
- Log đủ 3 lớp: request-level (latency, status), generation-level (token in/out), error-level (exception, stack trace).
- Xem được theo thời gian (hôm nay / 7 ngày) — tối thiểu là endpoint JSON, đẹp hơn thì dashboard Langfuse (free tier).
- Log **không chứa** nội dung tài liệu nhạy cảm quá mức cần thiết (lưu ID + trích đoạn ngắn, không lưu cả file).

**API/endpoint gợi ý:**
- `GET /api/admin/metrics?range=7d` — tổng request, p50/p95 latency, tổng token, token ước tính chi phí.
- Chưa cần build UI riêng — dùng Langfuse Cloud free + UptimeRobot là đủ (xem [04-tech-stack.md](04-tech-stack.md)).

**Cách test:**
- Tạo 20 request hỗn hợp (chat/RAG/vision) → metrics khớp số request thật ± 0.
- Cố tình gây lỗi 1 request → log lỗi có đủ thông tin chẩn đoán (không phải 500 chung chung).

## Liên kết

- [01 — Tầm nhìn & Scope](01-goals-and-scope.md) — các story nằm trọn trong 4 mục ✅ của scope
- [03 — Kiến trúc & luồng xử lý](03-architecture-and-flows.md) — backend xử lý từng story theo luồng nào
- [05 — Milestones](05-milestones.md) — story nào được nghiệm thu ở milestone nào
