# Tầm nhìn & Scope — GenAI Knowledge Assistant

> **Mục tiêu:** Hiểu rõ sản phẩm capstone "GenAI Knowledge Assistant" là gì, tại sao chọn đúng tầm nhìn này (vừa là sản phẩm thật vừa là công cụ học), và đặc biệt — tại sao giữ scope chặt lại là yếu tố quyết định dự án có hoàn thành hay không.
> Nguồn mở rộng từ: [08-FINAL-PROJECT](README.md)

---

## Tầm nhìn sản phẩm — một câu

> "Upload tài liệu của bạn → hỏi bất cứ điều gì → trả lời chính xác, có trích dẫn nguồn, kèm hiểu ảnh/hóa đơn."

Câu này gói gọn toàn bộ sản phẩm và cũng là **bài test nhanh** cho mọi quyết định sau này: một tính năng bất kỳ chỉ được làm nếu nó phục vụ đúng câu này. Cụ thể, câu này chứa 4 cam kết:

1. **Upload tài liệu của bạn** — dữ liệu là của người dùng (PDF, ảnh), hệ thống không đi cào web hay dùng dữ liệu công cộng. Điều này kéo theo bài toán ingestion: parse, chunk, embed, lưu trữ — chính là phần lõi của RAG.
2. **Hỏi bất cứ điều gì** — không giới hạn câu hỏi theo template; hệ thống phải xử lý câu hỏi tự nhiên, giữ ngữ cảnh hội thoại.
3. **Trả lời chính xác, có trích dẫn nguồn** — "chính xác" ở đây không có nghĩa là luôn đúng, mà là **không bịa**: mỗi câu trả lời trỏ được về đoạn tài liệu gốc. Đây là điểm phân biệt một chatbot demo với một công cụ đáng tin.
4. **Kèm hiểu ảnh/hóa đơn** — hệ thống không chỉ đọc text mà còn hiểu ảnh (VLM): hóa đơn/biểu mẫu → JSON có cấu trúc.

Sản phẩm được thiết kế để **xây dần theo lộ trình học** — mỗi giai đoạn là 1 tính năng. Nếu bạn muốn gắn vào dự án có sẵn trong workspace (ví dụ `doc-ocr` với MiniCPM-OCR, hay `deeptutor`), chỉ cần **đổi "miền dữ liệu"** — kiến trúc giữ nguyên. Đây là một quyết định thiết kế quan trọng: kiến trúc (ingest → embed → retrieve → rerank → generate + citation) hoàn toàn không phụ thuộc dữ liệu là PDF luật, slide bài giảng, hay tài liệu OCR. Đổi domain nghĩa là đổi tập tài liệu test và prompt ví dụ, không phải viết lại hệ thống.

## Mục tiêu kỹ thuật — đây là phần để học

Mục tiêu không chỉ là "có sản phẩm", mà là **học được gì từ việc xây nó**. Ba mục tiêu kỹ thuật:

- **RAG tự viết + eval có số liệu** (không phải RAG "chạy được là xong"). Tự viết nghĩa là tự làm ingestion, retrieval, reranking, generation — không bọc một framework low-code rồi hy vọng. Eval có số liệu nghĩa là có bộ câu hỏi test + metric (Ragas faithfulness, context recall) chạy được bằng một lệnh, để mỗi lần sửa prompt hay đổi chunk size đều biết mình tốt hơn hay tệ đi. Không có eval, mọi "tối ưu" chỉ là cảm giác.
- **Model chạy thật trên server CPU free** (không phụ thuộc API trả phí). Chạy local ép bạn đối mặt với giới hạn thực tế: latency trên CPU, quantization, quản lý RAM. Những giới hạn này dạy tư duy production nhiều hơn bất kỳ API nào — và giữ chi phí về 0đ.
- **Production-grade: logging, error handling, giám sát, chi phí đo được.** "Chạy được trên laptop" khác "production" ở ba chữ này. Một hệ thống production phải trả lời được: hôm nay có bao nhiêu request, lỗi bao nhiêu lần, mỗi câu trả lời tốn bao nhiêu token — cho dù toàn bộ tốn 0đ.

## Phạm vi (scope) — TUYỆT ĐỐI giữ trong phạm vi này

- ✅ **Upload PDF/ảnh → hỏi đáp có trích dẫn** — tính năng lõi, chính là luồng RAG.
- ✅ **Chatbot streaming, lịch sử hội thoại** — UX tối thiểu để dùng thật: trả lời từng token (streaming) và nhớ đoạn chat trước.
- ✅ **Trích xuất thông tin từ ảnh (hóa đơn/biểu mẫu → JSON)** — luồng VLM, biến ảnh thành dữ liệu có cấu trúc.
- ✅ **1 tài khoản user đơn giản** (hoặc không auth, private dùng) — nếu cần phân biệt người dùng thì auth tối giản; không cần cũng được, chạy private cho mình.

Ngoài phạm vi (❌): **Multi-tenant, thanh toán, mobile app, realtime collab.** Bốn thứ này mỗi thứ kéo theo cả một hệ thống con (phân quyền, billing, push notification, websocket scale-out…) nhưng không dạy thêm được kỹ năng AI nào — chúng là bài toán kỹ thuật phần mềm thuần, không phải bài toán AI. Nguyên tắc chung: capstone là nơi **chứng minh năng lực AI**, không phải nơi xây startup.

Khi có ý tưởng tính năng mới (và sẽ có), quy tắc là: **ghi vào backlog, không thêm vào capstone.** Xem chi tiết cách xử lý cám dỗ này ở [06-acceptance-and-risks.md](06-acceptance-and-risks.md) — rủi ro "quá ham tính năng mới".

## Tại sao tầm nhìn này đáng công sức

- **Đủ rộng** để đụng mọi kỹ năng cốt lõi của lộ trình: LLM internals (chạy model), RAG (toàn bộ pipeline), VLM (hiểu ảnh), Agents (tool ở M4), Deployment (lên VPS).
- **Đủ hẹp** để hoàn thành trong 4 tuần (M1–M4) bởi một người.
- **Có tiêu chí đo được** — "trả lời chính xác có trích dẫn" đo được bằng Ragas + citation check, không phải cảm tính. Xem [06-acceptance-and-risks.md](06-acceptance-and-risks.md).
- **Có giá trị thật** — sau khi hoàn thành, đây là công cụ bạn dùng hàng ngày cho tài liệu của chính mình, không phải demo vứt đi.

## Tại sao không phải là sản phẩm khác?

- **Không phải "chatbot trả lời chung chung"** — chatbot chung không có bài toán ingestion, retrieval, citation → không đụng tới kỹ năng RAG, và không có tiêu chí "chính xác" nào để đo.
- **Không phải "clone ChatGPT cho PDF"** — về mặt tính năng thì giống, nhưng mục đích khác hẳn: ở đây sản phẩm là **phương tiện học**, nên mọi thứ phải tự viết phần lõi (RAG tự viết + eval), không bọc SaaS có sẵn.
- **Không phải "trợ lý đa năng có agent"** — agent là một tool nhỏ ở M4, không phải trung tâm. Sản phẩm trung tâm là RAG + citation, vì đó là thứ khó nhất và đáng học nhất.

## Liên kết

- [02 — User Stories](02-user-stories.md) — tầm nhìn được cụ thể hóa thành các kịch bản người dùng
- [03 — Kiến trúc & luồng xử lý](03-architecture-and-flows.md) — tầm nhìn được hiện thực hóa bằng pipeline kỹ thuật
- [06 — Acceptance & rủi ro](06-acceptance-and-risks.md) — cách kiểm chứng "trả lời chính xác, có trích dẫn"
