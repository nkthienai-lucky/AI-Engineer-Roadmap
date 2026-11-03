# Tech Stack — chốt lại, không bàn thêm

> **Mục tiêu:** Nắm bảng công nghệ đã chốt cho capstone, hiểu **tại sao** mỗi lựa chọn được chọn (trade-off so với phương án thay thế), và hiểu ranh giới: "chốt rồi, không bàn thêm" — mỗi lần cân nhắc đổi stack là một lần rời xa việc hoàn thành sản phẩm.
> Nguồn mở rộng từ: [08-FINAL-PROJECT.md](../../../08-FINAL-PROJECT.md)

---

## Bảng tech stack (giữ nguyên từ spec)

| Lớp | Chọn | Lý do |
|---|---|---|
| Backend | **FastAPI + Python** | Async, streaming SSE, ecosystem AI |
| Frontend | **React** (hoặc Gradio nếu muốn nhanh) | SPA chat UI |
| LLM | **Qwen2.5-3B-Instruct Q4** qua Ollama | Tiếng Việt tốt, CPU OK |
| VLM | **SmolVLM2-2.2B** | Nhẹ, chạy CPU |
| Embedding | **Qwen3-Embedding-0.6B** | Đa ngôn ngữ, CPU mượt |
| Reranker | **bge-reranker-base** | CPU OK |
| Vector DB | **PostgreSQL + pgvector** | 1 service duy nhất, free |
| Parse | **Docling** | PDF lộn xộn → markdown |
| Eval | **Ragas** | Faithfulness / recall |
| Infra | Docker Compose trên Oracle VPS | Xem [07-DEPLOYMENT.md](../../../07-DEPLOYMENT.md) |
| Observability | **Langfuse Cloud free tier** + UptimeRobot | Self-host quá nặng với VPS 12GB (cần ClickHouse) — xem 06-ECOSYSTEM |

## Giải thích từng dòng — lý do & trade-off

### Backend — FastAPI + Python
Python là ngôn ngữ mặc định của toàn bộ hệ sinh thái AI (transformers, sentence-transformers, Ragas) — dùng Python nghĩa là không cần "cầu" sang ngôn ngữ khác. FastAPI hỗ trợ **async + streaming SSE** gốc: chat streaming từng token là yêu cầu UX bắt buộc (US3), và FastAPI làm điều đó với ít code hơn Flask (phải tự xử lý stream) hay Django (quá nặng cho 1 service API). Trade-off: FastAPI không có admin panel sẵn như Django — chấp nhận được vì admin chỉ cần 1 endpoint metrics (US5).

### Frontend — React (hoặc Gradio nếu muốn nhanh)
React cho SPA chat UI đẹp, kiểm soát được citation display (bấm vào trích dẫn → mở đoạn nguồn). **Gradio** là phương án dự phòng "nhanh": vài chục dòng là có UI chat + upload, hợp khi ưu tiên backend và deadline M1–M2 siết. Khi nào chọn Gradio? Khi frontend ăn quá nhiều thời gian — nhưng M4 nên có React để portfolio đẹp.

### LLM — Qwen2.5-3B-Instruct Q4 qua Ollama
Ba tiêu chí: **tiếng Việt tốt** (Qwen hỗ trợ đa ngôn ngữ mạnh trong phân khúc nhỏ), **chạy OK trên CPU** (3B tham số, quantize Q4_K_M ≈ 2GB RAM), và **miễn phí hoàn toàn** (model open, Ollama open). So với model lớn hơn (7B, 14B): nhanh hơn, nhẹ hơn, chấp nhận kém hơn về suy luận phức tạp — hợp với "hỏi đáp trên tài liệu" (context đã được retrieval nặn sẵn). Ollama lo nạp/tháo model, quantize, và expose API OpenAI-compatible — backend không đụng trực tiếp vào weights.

### VLM — SmolVLM2-2.2B
Nhẹ (2.2B), chạy được CPU, chuyên hiểu ảnh → text/caption. So với MiniCPM-OCR hay Qwen2-VL: SmolVLM2 tối ưu cho "nhỏ mà đủ" — hợp với bài toán hóa đơn/biểu mẫu (ảnh 1 lần, không cần VLM mạnh). Trade-off: độ chính xác trên ảnh phức tạp thấp hơn model lớn — bù bằng cách test 10 ảnh thật ngay từ M3, không đợi đến lúc deploy.

### Embedding — Qwen3-Embedding-0.6B
Đa ngôn ngữ (câu hỏi tiếng Việt, tài liệu có thể trộn Việt/Anh) và 0.6B nên encode hàng nghìn chunk nhanh trên CPU. Quan trọng: embedding model phải **cùng 1 model cho cả ingest lẫn query** — đổi model = phải embed lại toàn bộ kho (xem luồng RAG ở [03-architecture-and-flows.md](03-architecture-and-flows.md)).

### Reranker — bge-reranker-base
Cross-encoder chấm lại (câu hỏi, chunk) chính xác hơn embedding cosine — chi phí CPU thêm vài chục ms mỗi query, đổi lại top-5 context sạch hơn hẳn. "base" (chứ không phải large) vì rerank chỉ chạy trên 20 ứng viên mỗi query, không cần model lớn.

### Vector DB — PostgreSQL + pgvector
Quyết định "1 service duy nhất" đáng giá hơn mọi benchmark: trên VPS 12GB, mỗi service thêm là thêm RAM, thêm backup, thêm chỗ hỏng. pgvector kiêm luôn DB quan hệ (users, conversations, feedback) — không cần tách Qdrant/Milvus. Trade-off: pgvector chậm hơn vector DB chuyên dụng ở quy mô hàng triệu vector — nhưng kho tài liệu cá nhân chỉ hàng nghìn chunk, không phải vấn đề.

### Parse — Docling
PDF "lộn xộn" (bảng, cột, header/footer) → markdown sạch, giữ cấu trúc. Parse sai = hỏng toàn bộ pipeline downstream, nên đây là chỗ **không nên tiết kiệm**. So với PyPDF/pdfplumber (text thuần, mất bảng) hay Unstructured (nặng hơn): Docling cân bằng tốt nhất giữa chất lượng và độ nhẹ.

### Eval — Ragas
Đo **faithfulness** (có bịa không) và **context recall** (retrieval có tìm đúng chunk không) bằng số liệu — biến "hình như RAG tốt hơn" thành "faithfulness tăng từ 0.72 lên 0.84". Đây là công cụ biến mục tiêu kỹ thuật "eval có số liệu" thành hiện thực. Xem chi tiết metric tại folder [03-RAG](../03-RAG/06-evaluation-ragas.md).

### Infra — Docker Compose trên Oracle VPS
Compose biến cả cụm (nginx, FastAPI, Ollama, Postgres) thành 1 file `docker-compose.yml` — recreate được ở bất kỳ đâu, không "chạy được trên máy tôi". Chi tiết hạ tầng và cách deploy: [07-DEPLOYMENT.md](../../../07-DEPLOYMENT.md).

### Observability — Langfuse Cloud free tier + UptimeRobot
Langfuse trace từng request (prompt, context, latency, token) — trực tiếp nuôi US5. UptimeRobot ping định kỳ, cảnh báo khi site chết (điều kiện acceptance "7 ngày không crash").

**Ghi chú quan trọng — tại sao KHÔNG self-host Langfuse:** bản self-host của Langfuse cần **ClickHouse** (cùng Postgres, Redis, worker…) — ClickHouse một mình nó đã ngốn nhiều GB RAM và CPU, cộng với Ollama (LLM + VLM + embed + rerank) trên VPS 12GB là kịch bản chắc chắn OOM. Langfuse Cloud free tier cho đủ số trace của 1 dự án cá nhân, không tốn RAM VPS. Nếu sau này muốn self-host (khi có VPS mạnh hơn), tham khảo mục observability trong [06-ECOSYSTEM.md](../../../06-ECOSYSTEM.md).

## Quy tắc "chốt rồi, không bàn thêm"

Bảng này đóng lại tại thời điểm bắt đầu M1. Mọi đề xuất đổi stack (Qdrant thay pgvector, Llama thay Qwen…) ghi vào backlog và **chỉ xét lại sau khi capstone xong** — mỗi lần đổi stack là vài ngày làm lại ingest + test, mà không đổi được kết quả học được gì.

## Liên kết

- [03 — Kiến trúc & luồng xử lý](03-architecture-and-flows.md) — các thành phần này ghép với nhau thế nào
- [05 — Milestones](05-milestones.md) — công cụ nào cần cho milestone nào
- [06 — Acceptance & rủi ro](06-acceptance-and-risks.md) — rủi ro RAM/OOM gắn với đúng các thành phần ở đây
