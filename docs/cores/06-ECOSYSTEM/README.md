# 06 — Bảng tra nhanh hệ sinh thái GenAI

> Tài liệu tham khảo — tra khi cần chọn công cụ. Không cần học thuộc!

---

## 🗄️ Vector DB

| Vector DB | Loại | Tự host free | Cloud free tier | Hợp với |
|---|---|---|---|---|
| **pgvector** | Extension Postgres | ✅ (cùng Postgres) | Supabase / Neon free | **Mặc định nên dùng** — không thêm service, dễ deploy |
| **Qdrant** | Purpose-built (Rust) | ✅ Docker, nhẹ | 1GB cluster, 4 vCPU | Performance cao, filter mạnh |
| **Chroma** | Embedded, nhẹ | ✅ 1 dòng cài | Có managed mới | Học tập, prototype notebook |
| **Weaviate** | Purpose-built | ✅ Docker | Sandbox 14 ngày | Hybrid search (text + vector) |
| **Milvus / Zilliz** | Scale lớn | ✅ (nặng) | Zilliz: 5GB free | Hàng triệu vectors |
| **FAISS** | Library (không server) | ✅ | ❌ | Nghiên cứu, tìm hiểu thuật toán |
| **Pinecone** | Managed SaaS | ❌ | Free tier có hạn | Không muốn tự quản lý |

**Chọn nhanh:** Học → `Chroma`/`FAISS`. Deploy Oracle VPS → `pgvector`. Production lớn → `Qdrant`/`Weaviate`.

🔗 So sánh chi tiết: [buttondown.com/ultradune/archive/eval-002-vector-databases-in-2026](https://buttondown.com/ultradune/archive/eval-002-vector-databases-in-2026-qdrant-vs/) · [aitoolguide.ai/blog/best-vector-databases-for-rag-2026](https://aitoolguide.ai/blog/best-vector-databases-for-rag-2026/)

---

## 🧲 Embedding Models

| Model | Open source | Chất lượng | Chạy CPU? | Ghi chú |
|---|---|---|---|---|
| **Qwen3-Embedding-0.6B** | ✅ Apache 2.0 | Rất tốt | ✅ Mượt | **Best pick cho CPU / tiếng Việt** |
| **BGE-M3** (BAAI) | ✅ | Tốt, đa ngôn ngữ | ✅ OK | Đa ngôn ngữ mạnh |
| **nomic-embed-text-v1.5** | ✅ | Tốt | ✅ Rất nhẹ | Hay dùng với Ollama |
| Qwen3-Embedding-8B | ✅ | Top đầu | ⚠️ Cần GPU | Khi có GPU |
| OpenAI text-embedding-3 | ❌ | Tốt | — | $0.13/1M tokens |

🔗 Xếp hạng live: [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard)

---

## 🔁 Rerankers

| Tool | Loại | Ghi chú |
|---|---|---|
| **bge-reranker-base/v2-m3** | Cross-encoder, open source | Chạy CPU OK — khuyến nghị |
| FlashRank | Nhẹ, CPU-first | Nhanh, chất lượng khá |
| Cohere Rerank | API trả phí | Chất lượng cao, tính theo call |

---

## 🧩 Pipeline RAG — công cụ theo mắt xích

| Mảnh | Công cụ | Ghi chú |
|---|---|---|
| Parse | **Docling**, Unstructured, LlamaParse | Docling (IBM) rất tốt cho PDF lộn xộn |
| Chunking | tiktoken + chiến lược thủ công | Quan trọng hơn tool — thử 256/512/1024 |
| Orchestration | LangChain / LlamaIndex / Haystack | Biết 1 trong 2; học pipeline trước, framework sau |
| Agent | **LangGraph**, function-calling thuần | LangGraph cho agent có trạng thái |
| Eval | **Ragas**, DeepEval, TruLens | Ragas: faithfulness, context recall |
| Observability | Langfuse (Cloud free tier — khuyên dùng), LangSmith | ⚠️ Langfuse self-host v3 cần ClickHouse + Redis + Postgres — **quá nặng** với VPS 12GB, hãy dùng Cloud free hoặc tách VPS riêng |
| Guardrails | NeMo Guardrails, Guardrails AI | Chống prompt injection, lọc output |
| Serving | **Ollama**, llama.cpp, vLLM (GPU) | Ollama cho dev, llama.cpp cho production CPU |
| Caching | GPTCache, semantic cache (Redis) | Giảm chi phí khi query lặp |

🔗 Khám phá thêm: [awesome-rag-production](https://yigtwxx.github.io/awesome-rag-production/) · [llmknowledge.org/appendix](https://llmknowledge.org/appendix)

---

## 🤖 Model chọn nhanh (chạy CPU / server nhỏ)

| Việc | Model đề nghị | Ghi chú |
|---|---|---|
| Chat tiếng Việt | **Qwen3.5-9B Q4** | Benchmark 11/2026 trên RTX 5060: 42 tok/s — điểm ngọt chatbot |
| Chat nhẹ nhất | Qwen2.5-0.5B/1.5B Q4 | Rất nhanh trên CPU |
| VLM (ảnh) | **Gemma 4 E4B** | Đa nhiệm ảnh + audio; OCR tiếng Việt chính xác nhất trong test |
| OCR/VLM tài liệu | **GLM-OCR**, MiniCPM-V, Qwen-VL | Xem thêm dự án `doc-ocr` trong workspace |
| Embedding | Qwen3-Embedding-0.6B | Xem bảng trên |

Nguồn model: [huggingface.co](https://huggingface.co/) (tìm bản GGUF quantized).
Chi tiết: [05-model-quick-picks.md](05-model-quick-picks.md) (chọn model + tối ưu inference) · [07-tech-stack-lifecycle.md](07-tech-stack-lifecycle.md) (tech stack toàn lifecycle).

---

## 💰 Hạ tầng tính tiền (khi nào free không đủ)

| Nhu cầu | Nơi rẻ | Giá tham khảo |
|---|---|---|
| GPU theo giờ (train/batch) | **Vast.ai** marketplace | RTX 4090 từ ~$0.13–0.51/h |
| GPU ổn định hơn | **RunPod Community** | RTX 4090 $0.34/h ≈ $250/tháng |
| Inference API theo request | **RunPod Serverless / Modal** | Pay-per-request, ~$5–20/tháng app nhỏ |
| Rẻ nhất, chấp nhận rủi ro | SaladCloud | Từ $0.02/h (hay disconnect) |

🔗 Giá live: [gpudiff.com](https://gpudiff.com/) · [gputable.dev](https://gputable.dev/) · [gpucloudprices.com](https://gpucloudprices.com/)
