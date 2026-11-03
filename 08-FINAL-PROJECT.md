# 08 — Spec sản phẩm cuối (Capstone)

> **Sản phẩm:** "GenAI Knowledge Assistant" — trợ lý tri thức cá nhân/doanh nghiệp nhỏ:
> chat hỏi đáp trên tài liệu riêng (RAG) + hiểu ảnh (VLM) + có agent, deploy **0đ hoàn toàn**.
>
> 💡 Sản phẩm này được thiết kế để **xây dần theo lộ trình học** — mỗi giai đoạn là 1 tính năng.
> Nếu bạn muốn gắn vào dự án có sẵn trong workspace (ví dụ `doc-ocr` với MiniCPM-OCR, hay `deeptutor`),
> chỉ cần đổi "miền dữ liệu" — kiến trúc giữ nguyên.

---

## 1. 🎯 Tầm nhìn & mục tiêu

**Một câu:** "Upload tài liệu của bạn → hỏi bất cứ điều gì → trả lời chính xác, có trích dẫn nguồn, kèm hiểu ảnh/hóa đơn."

**Mục tiêu kỹ thuật (đây là phần để học, không chỉ để có sản phẩm):**
- RAG tự viết + eval có số liệu (không phải RAG 'chạy được là xong')
- Model chạy thật trên server CPU free (không phụ thuộc API trả phí)
- Production-grade: logging, error handling, giám sát, chi phí đo được

**Phạm vi (scope) — TUYỆT ĐỐI giữ trong phạm vi này:**
- ✅ Upload PDF/ảnh → hỏi đáp có trích dẫn
- ✅ Chatbot streaming, lịch sử hội thoại
- ✅ Trích xuất thông tin từ ảnh (hóa đơn/biểu mẫu → JSON)
- ✅ 1 tài khoản user đơn giản (hoặc không auth, private dùng)
- ❌ Multi-tenant, thanh toán, mobile app, realtime collab — **ngoài phạm vi**

---

## 2. 👤 User stories (tối thiểu)

1. *Là người dùng,* tôi upload 1 bộ tài liệu PDF → hỏi câu hỏi → nhận câu trả lời kèm **trích dẫn nguồn** (trang/đoạn).
2. *Là người dùng,* tôi upload ảnh hóa đơn → hệ thống trả về JSON có cấu trúc (số tiền, ngày, nhà cung cấp...).
3. *Là người dùng,* tôi chat bình thường (không cần tài liệu) → model trả lời, giữ ngữ cảnh hội thoại.
4. *Là người dùng,* tôi thấy câu trả lời không đúng → bấm "không đúng" → hệ thống ghi nhận (dữ liệu để cải thiện).
5. *Là người vận hành,* tôi xem được log: bao nhiêu request, latency, chi phí token ước tính.

---

## 3. 🏗️ Kiến trúc

```
                        ┌──────────────────────────────────────┐
User ──HTTPS──► Cloudflare ──►│  Oracle Always Free VPS (2CPU/12GB) │
                        │                                      │
                        │  nginx ──► FastAPI backend            │
                        │             │                         │
                        │   ┌─────────┼──────────────┐          │
                        │   ▼         ▼              ▼          │
                        │ Ollama   pgvector     SmolVLM2        │
                        │ (Qwen    (Postgres)   (ảnh → text)    │
                        │ 2.5-3B)      │                         │
                        │   ▲     ┌────┴─────┐                  │
                        │   │     ▼          ▼                  │
                        │   │  bge-embed  bge-reranker          │
                        │   │  0.6B       (CPU)                 │
                        └───┼──────────────────────────────────┘
                            │
                     Langfuse (trace, optional)
```

**Luồng RAG:** `PDF → Docling parse → chunk 512 (overlap 64) → embed (Qwen3-Embed-0.6B) → pgvector → [query embed → top-20 → rerank top-5] → prompt + context → Qwen2.5-3B → trả lời + citation`

**Luồng ảnh:** `ảnh → SmolVLM2-2.2B → caption/JSON → (tuỳ chọn) đưa vào RAG context`

## 4. 🧱 Tech stack (chốt lại, không bàn thêm)

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
| Infra | Docker Compose trên Oracle VPS | Xem [07-DEPLOYMENT.md](07-DEPLOYMENT.md) |
| Observability | **Langfuse Cloud free tier** + UptimeRobot | Self-host quá nặng với VPS 12GB (cần ClickHouse) — xem 06-ECOSYSTEM |

## 5. 📅 Milestones (ghép với lộ trình học)

| Milestone | Tương ứng | Nội dung | Nghiệm thu |
|---|---|---|---|
| **M1** (tuần 15) | GĐ1–2 xong | Chat core: FastAPI + Ollama, streaming, history | Chat hoạt động trên web |
| **M2** (tuần 16) | GĐ3 xong | RAG: ingest PDF → hỏi đáp có citation | Trích dẫn đúng trang; Ragas faithfulness > 0.8 |
| **M3** (tuần 17) | GĐ4 xong | VLM: upload ảnh → mô tả / trích JSON | Trích xuất đúng 8/10 hóa đơn test |
| **M4** (tuần 18) | GĐ5 xong | Production: deploy Oracle, monitoring, agent tool (tìm kiếm web/tính toán) | Site live trên domain, uptime > 99% tuần đầu |

**Quy tắc:** M1–M3 chạy local + HF Spaces; M4 mới deploy production. Không mở rộng scope ngoài 4 milestone.

## 6. ✅ Tiêu chí nghiệm thu cuối (acceptance criteria)

- [ ] **Live:** truy cập bằng domain (is-a.dev hoặc domain riêng), HTTPS hoạt động
- [ ] **RAG đúng:** 30 câu hỏi test → faithfulness > 0.8, context recall > 0.7 (Ragas)
- [ ] **Citation:** mỗi câu trả lời RAG trỏ được về đoạn nguồn gốc
- [ ] **VLM:** 10 ảnh hóa đơn/biểu mẫu → trích JSON đúng ≥ 8
- [ ] **Performance:** chat trả lời token đầu < 3s (CPU), streaming mượt
- [ ] **Stability:** chạy 7 ngày không crash; có UptimeRobot + log lỗi (Sentry/Langfuse)
- [ ] **Chi phí = 0đ:** mọi thành phần nằm trong danh sách free đã chốt
- [ ] **Portfolio:** README + demo video ≤ 5 phút + bài viết "học được gì"

## 7. ⚠️ Rủi ro & cách xử lý

| Rủi ro | Xử lý |
|---|---|
| Model 3B trả lời chậm trên 2 OCPU | Quantize Q4_K_M, giới hạn max_tokens, cache semantic cho câu lặp |
| 12GB RAM không đủ (LLM + VLM + DB) | Tải VLM theo demand (unload khi không dùng); hoặc tách VLM sang 1 VPS AMD nhỏ |
| Oracle "out of capacity" khi tạo VPS | Thử region Singapore/Osaka, thử lại nhiều lần; xem [07-DEPLOYMENT.md](07-DEPLOYMENT.md) |
| RAG trả lời sai/hallucination | Bắt buộc citation + guardrail "chỉ trả lời dựa trên context"; theo dõi feedback user (US4) |
| Cloudflare cắt stream (524) khi model sinh lâu | SSE + keepalive ping 15–30s + giới hạn max_tokens — xem 07-DEPLOYMENT mục C |
| OOM kill khi load cả LLM + VLM | OLLAMA_MAX_LOADED_MODELS=1, swap 4GB, unload VLM khi không dùng — xem 07-DEPLOYMENT mục D |
| Quá ham tính năng mới | Quay lại đọc "Phạm vi" — mọi request mới ghi vào backlog, không thêm vào capstone |

## 8. 📈 Sau khi hoàn thành (không thuộc lộ trình)

- Thêm auth + multi-user (Supabase Auth free)
- Fine-tune LoRA trên feedback user đã ghi nhận (US4)
- Chuyển LLM lên GPU khi có ngân sách (RunPod serverless — chỉ tốn khi có request)
- Benchmark so sánh với RAG-as-a-service (để biết giá trị tự build)

---

## ✅ Tóm lại — "chốt" hướng đi

| | Quyết định |
|---|---|
| **Học** | 5 giai đoạn, 18 tuần, mỗi giai đoạn 1 project (xem [README.md](README.md)) |
| **Cốt lõi phải giỏi** | Transformer (tự code), RAG + **Eval**, deploy production |
| **Công cụ mặc định** | Ollama · pgvector · Qwen3-Embed · Ragas · FastAPI · Docker |
| **Sản phẩm cuối** | GenAI Knowledge Assistant (chat + RAG + VLM), 4 milestone |
| **Hạ tầng** | Oracle Always Free (Singapore/Osaka) + Cloudflare — **0đ** |
| **Không làm** | Framework-first, học mọi thứ, mở rộng scope, chờ "học đủ rồi mới build" |
