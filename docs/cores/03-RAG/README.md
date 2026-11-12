# Giai đoạn 3 — RAG End-to-End (Tuần 8–11)

> **Mục tiêu:** Xây hệ thống hỏi đáp trên tài liệu riêng, **có đánh giá chất lượng bằng số liệu**. RAG là kỹ năng được săn đón nhiều nhất hiện tại cho AI Engineer.
> **Đầu ra:** Docs-QA app (chat với PDF/website, trích dẫn nguồn) + báo cáo eval.

---

## 🎓 Nội dung cần học

### Tuần 8 — Embedding & Vector Search
- Embedding model: hoạt động, chọn model thế nào (MTEB, đa ngôn ngữ, tiếng Việt)
- Vector similarity: cosine, dot product, L2
- Vector DB: **thành thạo 1 trong 2** — `pgvector` (khuyên dùng) hoặc `Qdrant`
- ANN (Approximate Nearest Neighbor): HNSW, IVF — hiểu khái niệm

### Tuần 9 — Ingestion & Chunking (nửa quan trọng nhất!)
- Parse tài liệu: PDF, Word, HTML (`Docling`, `Unstructured`)
- Chiến lược chunking: fixed-size, overlap, semantic chunking, theo heading
- Metadata: nguồn, trang, thời gian — dùng để filter + trích dẫn
- Indexing pipeline: document → chunk → embed → store

### Tuần 10 — Retrieval & Generation
- Naive RAG → cải thiện: query rewriting, HyDE, multi-query
- **Reranking**: cross-encoder (`bge-reranker`), tại sao cần (top-k retrieval ≠ top-k đúng)
- Hybrid search: BM25 + vector (RRF fusion)
- Generation với context + trích dẫn nguồn (citation)
- Prompt chống hallucination: "chỉ trả lời dựa trên context"

### Tuần 11 — **EVALUATION** (phần quan trọng nhất!)
- Metrics: **faithfulness**, answer relevancy, context precision/recall
- Framework: **Ragas** (bắt buộc), DeepEval, TruLens
- Xây eval dataset: câu hỏi + ground truth
- Đọc kết quả eval → sửa pipeline (chunk size? rerank? embedding?) → đo lại
- Advanced: Agentic RAG, GraphRAG (đọc khái niệm)

---

## 📚 Tài nguyên

| Tài nguyên | Dùng để | Link |
|---|---|---|
| **LangChain — RAG From Scratch** (video series) | RAG từng bước, không black-box | [youtube playlist](https://www.youtube.com/watch?v=bsndTzVun4w&list=PLFAIKu28IekQTCE_JHCVkDt9H0eR1ph36) |
| **DeepLearning.AI — RAG courses** | Các khóa ngắn: Advanced RAG, Retrieval | [deeplearning.ai/resources/generative-ai-courses-guide](https://www.deeplearning.ai/resources/generative-ai-courses-guide) |
| **LlamaIndex docs** | Framework RAG (tham khảo khi build thật) | [docs.llamaindex.ai](https://docs.llamaindex.ai/) |
| **Ragas docs** | Đánh giá RAG | [docs.ragas.io](https://docs.ragas.io/) |
| **MTEB Leaderboard** | Chọn embedding model | [hf.co/spaces/mteb/leaderboard](https://huggingface.co/spaces/mteb/leaderboard) |
| **Bảng so sánh Vector DB / embedding / free tier** | Tra nhanh | [06-ECOSYSTEM](../06-ECOSYSTEM/README.md) |

---

## 🛠️ Project GĐ3 — "Docs-QA có trích dẫn + Eval"

**Yêu cầu:**
1. **Phiên bản 1 (tuần 9):** RAG pipeline **viết tay, KHÔNG dùng LangChain/LlamaIndex** — chỉ dùng embedding API + pgvector/Qdrant + LLM API
2. **Phiên bản 2 (tuần 10):** Thêm reranker + hybrid search
3. **Phiên bản 3 (tuần 11):** Eval với Ragas trên 30+ câu hỏi test; in bảng so sánh trước/sau cải tiến
4. UI web (Gradio/Streamlit): upload PDF → hỏi → trả lời kèm nguồn (trang X, đoạn Y)

**Tiêu chí hoàn thành (DoD):**
- [ ] Pipeline tay chạy được end-to-end (không framework)
- [ ] Trả lời có trích dẫn nguồn, click ra được đoạn gốc
- [ ] Bảng eval Ragas: faithfulness > 0.8, context recall > 0.7
- [ ] Có bảng so sánh "naive vs +rerank vs +hybrid" bằng số liệu
- [ ] Deploy: **Render free** nếu app nhẹ (không cần DB server trên cloud) hoặc **Oracle VPS** nếu cần pgvector chạy 24/7 *(HF Spaces đã hết free cho Gradio từ 2026)*

---

## ⚠️ Lỗi thường gặp
- Dùng LangChain từ đầu → không hiểu pipeline, không debug được. **Viết tay trước, framework sau.**
- Quên eval → sửa pipeline trong mù, không biết cải tiến có thật sự tốt hơn
- Chunk size để mặc định (1024) → retrieval kém. Thử 256/512/1024 và **đo** sự khác biệt
- Chỉ tối ưu generation prompt, bỏ qua retrieval → 90% vấn đề RAG nằm ở retrieval

---

## 🔗 Chuyển tiếp
Xong GĐ3 → bạn có kỹ năng "hái ra tiền" nhất. GĐ4 ([04-VLM](../04-VLM/README.md)) mở rộng sang ảnh — multimodal là xu hướng hiện tại.
