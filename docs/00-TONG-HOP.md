# 00 — TỔNG HỢP: Toàn cảnh AI Engineer Roadmap (18 tuần)

> Đây là "bản đồ" để nhìn thấy **toàn bộ** trước khi đi vào từng phần. Khi lạc, quay lại file này.
> Chi tiết học từ đầu: [cores/01](cores/01-FOUNDATIONS/) → [cores/02](cores/02-LLM-INTERNALS/) → [cores/03](cores/03-RAG/) → [cores/04](cores/04-VLM/) → [cores/05](cores/05-AGENTS-PRODUCTION/); tra cứu: [cores/06](cores/06-ECOSYSTEM/) · deploy: [cores/07](cores/07-DEPLOYMENT/) · capstone: [cores/08](cores/08-FINAL-PROJECT/).

---

## 1. Lộ trình trong 1 hình dung

```
01 Nền tảng        DÙNG ĐƯỢC GenAI        gọi API, prompt, chatbot CLI, deploy demo
02 Bản chất LLM    HIỂU BẢN CHẤT          transformer, attention, tự train mini-GPT
03 RAG             KỸ NĂNG "HÁI RA TIỀN"  hỏi đáp dữ liệu riêng + eval có số liệu
04 VLM             MỞ RỘNG SANG ẢNH       multimodal, OCR, trích xuất tài liệu
05 Agents + Prod   THÀNH SẢN PHẨM         agent, guardrails, observability, deploy
```

**Sợi chỉ đỏ xuyên suốt:** mỗi giai đoạn = 1 project chạy được → cuối cùng ghép thành **Capstone** ([cores/08](cores/08-FINAL-PROJECT/)) — "GenAI Knowledge Assistant": chat + RAG + VLM + agent, deploy 0đ.

## 2. Ba trục kiến thức (hiểu theo trục, đừng học theo tuần)

| Trục | Nằm ở đâu | Cốt lõi |
|---|---|---|
| **Hiểu model** | 01 → 02 → 04 | token/embedding → attention/transformer → ViT/CLIP/VLM. Mục tiêu: không phải "người dùng API" — tự code được attention, giải thích được KV cache, quantization |
| **Retrieval & chất lượng** | 03 (mở rộng ở 08) | chunking → embedding/vector DB → rerank/hybrid → **eval (Ragas)**. 90% vấn đề RAG nằm ở retrieval, và không eval = sửa trong mù |
| **Production & vận hành** | 05 + 07 | streaming SSE, guardrails, observability, chi phí/token, RAM budget, firewall, backup. Sống còn với hạ tầng 12GB free |

## 3. Bốn quy tắc "chốt" phải nhớ
1. **Viết tay trước, framework sau** (RAG không LangChain từ đầu) — không hiểu pipeline thì không debug được.
2. **Eval bằng số liệu** — faithfulness > 0.8, context recall > 0.7; mọi "cải tiến" phải đo trước/sau.
3. **Local trước, cloud sau** — Ollama từ tuần 3 để hiểu RAM/VRAM/tốc độ; deploy server thật ở milestone cuối, không "chạy máy mình là xong".
4. **Scope là luật** — capstone đóng băng ở 4 milestone, tính năng mới vào backlog.

## 4. Tech stack chốt (0đ, CPU)

| Lớp | Chọn | Vì sao |
|---|---|---|
| LLM chat | Qwen2.5-3B-Instruct Q4 (Ollama) | Tiếng Việt tốt, CPU OK |
| VLM | SmolVLM2-2.2B | Nhẹ, CPU chạy được |
| Embedding | Qwen3-Embedding-0.6B | Đa ngôn ngữ, CPU mượt |
| Reranker | bge-reranker-base | CPU OK |
| Vector DB | PostgreSQL + pgvector | 1 service duy nhất, free |
| Parse / Eval | Docling / Ragas | PDF lộn xộn / faithfulness+recall |
| Backend / Deploy | FastAPI + Docker Compose trên Oracle Always Free + Cloudflare | Streaming SSE, 0đ |

## 5. Timeline & đầu ra mỗi chặng

| Tuần | GĐ | Project đầu ra | DoD tóm gọn |
|---|---|---|---|
| 1–3 | [01](cores/01-FOUNDATIONS/) | Chatbot CLI + Gradio demo | Stream mượt, đổi backend = 1 dòng, URL public |
| 4–7 | [02](cores/02-LLM-INTERNALS/) | Mini-GPT | Attention tự viết ~50 dòng, loss giảm, giải thích được |
| 8–11 | [03](cores/03-RAG/) | Docs-QA + eval | Trích dẫn nguồn, Ragas đạt ngưỡng, bảng naive vs rerank vs hybrid |
| 12–14 | [04](cores/04-VLM/) | Trợ lý ảnh | So sánh 2 VLM trên 20 ảnh, < 15s/ảnh CPU |
| 15–18 | [05](cores/05-AGENTS-PRODUCTION/) | Capstone (M1→M4) | Live domain, uptime > 99%, chi phí 0đ |

Song song suốt lộ trình: deploy từng project lên web ([cores/07](cores/07-DEPLOYMENT/)).

## 6. Cách học tối ưu (rút từ "lỗi thường gặp" của cả 5 GĐ)
- **Gõ code theo Karpathy**, không xem thụ động — nền tảng GĐ2 quyết định giới hạn trên của bạn.
- Mỗi project: làm bản "viết tay" trước → mới nâng cấp (rerank, hybrid, framework).
- Mỗi thay đổi pipeline = 1 lần đo (eval dataset 30+ câu).
- Học đến đâu deploy đến đó — lỗi production (524, OOM, firewall) chỉ học được khi deploy thật.
- [cores/06](cores/06-ECOSYSTEM/) là bảng **tra**, không học thuộc.

## 7. Mục tiêu cuối cùng (checklist hoàn thành)
- [ ] 5 project chạy được + 1 capstone trên portfolio (README + demo video ≤ 5 phút)
- [ ] Tự code transformer, tự xây RAG có eval, tự deploy production 0đ
- [ ] Trả lời được phỏng vấn: attention O(n²)? KV cache? faithfulness là gì? chunk size chọn sao?
