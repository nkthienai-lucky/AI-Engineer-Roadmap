# 04 — Pipeline RAG: công cụ theo mắt xích

> **Mục tiêu**: nắm vai trò từng mắt xích trong luồng RAG tổng thể (Parse → Chunking → Orchestration → Agent → Eval → Observability → Guardrails → Serving → Caching) và biết công cụ nào cho mắt xích nào — bảng tra, không học thuộc.
>
> Nguồn mở rộng từ: [06-ECOSYSTEM.md](../../../06-ECOSYSTEM.md)

---

## Bảng tra nhanh

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

---

## Vai trò từng mắt xích trong luồng RAG

1. **Parse** — biến PDF/DOCX/ảnh thành text có cấu trúc. PDF "lộn xộn" (cột, bảng, header) là nơi pipeline hỏng nhiều nhất: **Docling (IBM)** rất tốt cho loại này; Unstructured và LlamaParse là lựa chọn thay thế (LlamaParse mạnh với bảng phức tạp). Sai ở đây là sai suốt downstream.
2. **Chunking** — cắt text thành mảnh để embed. **Quan trọng hơn tool**: chiến lược thủ công (theo heading/đoạn, kèm metadata) thắng tool tự động; dùng `tiktoken` để đếm token và thử kích thước 256/512/1024 rồi so kết quả. Chunk quá nhỏ mất ngữ cảnh, quá lớn loãng embedding.
3. **Retrieval + Embedding** — vector DB + embedding model (xem 2 file liên quan dưới cùng folder). Trả về top-k.
4. **(Rerank)** — cross-encoder chấm lại top-k trước khi đưa vào LLM (xem [03-rerankers.md](03-rerankers.md)).
5. **Orchestration** — framework nối các bước trên thành pipeline. **LangChain / LlamaIndex / Haystack**: biết 1 trong 2 là đủ; nguyên tắc: **học pipeline trước, framework sau** — hiểu từng bước rồi mới dùng framework để khỏi bị "magic" che logic.
6. **Agent** — khi hệ thống phải gọi tool/nhiều bước có trạng thái: **LangGraph** cho agent có trạng thái (state machine rõ ràng); function-calling thuần của LLM đủ khi chỉ 1–2 tool đơn giản.
7. **Eval** — đo chất lượng thay vì cảm tính: **Ragas** (faithfulness = câu trả lời có bám context không, context recall = context đúng có được retrieve không), DeepEval, TruLens. Eval là mắt xích để biết chunk size/model nào tốt hơn.
8. **Observability** — log trace từng request để debug: **Langfuse (Cloud free tier — khuyên dùng)**, LangSmith. ⚠️ **Langfuse self-host v3 cần ClickHouse + Redis + Postgres — quá nặng với VPS 12GB**: hãy dùng Cloud free hoặc tách VPS riêng, đừng cài self-host lên server nhỏ đã chạy app.
9. **Guardrails** — **NeMo Guardrails, Guardrails AI**: chống prompt injection, lọc output (PII, nội dung cấm) trước khi trả user.
10. **Serving** — chạy model: **Ollama** cho dev (1 lệnh), **llama.cpp** cho production CPU (nhẹ, kiểm soát được), **vLLM** khi có GPU (throughput cao, batching).
11. **Caching** — **GPTCache, semantic cache (Redis)**: cache cả query tương tự ngữ nghĩa (không chỉ exact match) → giảm chi phí/tăng tốc khi query lặp.

## Thứ tự xây dựng gợi ý (đỡ rối khi mới)

- Làm đúng **Parse → Chunking → Retrieval** trước (chất lượng đầu vào quyết định phần lớn kết quả) — chưa cần framework.
- Thêm **Eval** (Ragas) sớm để có số liệu so sánh khi thử chunk size 256/512/1024 hoặc đổi model — không eval thì mọi chỉnh sửa chỉ là cảm tính.
- Khi pipeline chạy ổn mới thêm **Orchestration** để code gọn, rồi **Rerank/Guardrails/Caching** theo nhu cầu thật.
- **Observability** (Langfuse Cloud free) nên có từ lúc có user đầu tiên — trace giúp debug lỗi "hôm qua còn được hôm nay hỏng".
- Nguyên tắc xuyên suốt từ bảng gốc: **học pipeline trước, framework sau** — framework (LangChain/LlamaIndex/Haystack) là công cụ nối bước, không thay thế hiểu biết từng bước.

---

## Liên kết chi tiết (trong nguồn)

- Khám phá thêm: [awesome-rag-production](https://yigtwxx.github.io/awesome-rag-production/) · [llmknowledge.org/appendix](https://llmknowledge.org/appendix)

## Liên kết

- [01-vector-db.md](01-vector-db.md) · [02-embedding-models.md](02-embedding-models.md) — mắt xích retrieval trong bảng trên.
- [03-rerankers.md](03-rerankers.md) — bước rerank giữa retrieval và generation.
- [05-model-quick-picks.md](05-model-quick-picks.md) — model nào chạy được với Serving (Ollama/llama.cpp).
- [03-RAG.md](../../../03-RAG.md) — chương RAG gốc trong roadmap (lý thuyết pipeline đầy đủ).
