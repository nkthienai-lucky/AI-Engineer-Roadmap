# Tài nguyên & Lỗi thường gặp (Tuần 8–11)

> **Mục tiêu:** Tổng hợp tài nguyên học tập chính thức của GĐ3 (giữ nguyên link từ nguồn) và mở rộng 4 lỗi thường gặp — hiểu **tại sao** từng lỗi xảy ra và cách tránh cụ thể.
> Nguồn mở rộng từ: [03-RAG](README.md)

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

**Cách dùng gợi ý (tự nghĩ, không phải link mới):**

- **LangChain — RAG From Scratch:** xem **trước khi** viết V1 project — series này code từng bước (load → split → embed → retrieve → generate → evaluate) đúng theo pipeline viết tay, nên bạn sẽ thấy mọi khái niệm trong folder này hiện ra bằng code. Xem song song [03 — Ingestion](03-ingestion-parsing-chunking.md) và [04 — Retrieval](04-retrieval-reranking-hybrid.md).
- **DeepLearning.AI — RAG courses:** các khóa ngắn về Advanced RAG, Retrieval — dùng để đi sâu phần nâng cao (rerank, hybrid, eval) sau khi đã chạy được pipeline cơ bản.
- **LlamaIndex docs:** khi build thật và muốn năng suất cao hơn viết tay — đọc **sau** V1 để biết framework giải quyết đúng những gì bạn đã tự làm.
- **Ragas docs:** tài liệu chính cho [06 — Evaluation](06-evaluation-ragas.md) — metric definitions, cách build eval dataset, ví dụ code.
- **MTEB Leaderboard:** chọn embedding model theo điểm Retrieval + hỗ trợ tiếng Việt (xem [01 — Embedding](01-embedding-and-vector-search.md)).
- **Bảng so sánh Vector DB / embedding / free tier** trong [06-ECOSYSTEM](../06-ECOSYSTEM/README.md) — tra nhanh khi chọn pgvector/Qdrant và model.

## Lộ trình học gợi ý theo tuần (tổng hợp từ nguồn)

Gợi ý cách phân bổ tài nguyên cho từng tuần — sắp xếp từ nội dung Tuần 8–11 trong nguồn, không thêm tài liệu ngoài danh sách trên:

| Tuần | Nội dung (từ [03-RAG](README.md)) | Đọc trong folder này | Tài nguyên chính |
|---|---|---|---|
| 8 | Embedding & Vector Search | [01](01-embedding-and-vector-search.md), [02](02-vector-db-pgvector-qdrant.md) | MTEB Leaderboard, [06-ECOSYSTEM](../06-ECOSYSTEM/README.md) |
| 9 | Ingestion & Chunking (nửa quan trọng nhất!) | [03](03-ingestion-parsing-chunking.md) | LangChain RAG From Scratch |
| 10 | Retrieval & Generation | [04](04-retrieval-reranking-hybrid.md), [05](05-generation-citation-anti-hallucination.md) | DeepLearning.AI RAG courses |
| 11 | **Evaluation** (quan trọng nhất!) | [06](06-evaluation-ragas.md), [07](07-project-docs-qa.md) | Ragas docs |

Nhịp học khuyến nghị (tự nghĩ): mỗi tuần **học khái niệm vào đầu tuần, code vào cuối tuần**, và luôn chạy eval trên project song song — đừng đợi tuần 11 mới đo (lỗi số 2 dưới đây).

Mục tiêu cuối GĐ3: bạn không chỉ “dùng được RAG” mà **hiểu, đo và cải thiện được từng khâu** — đúng tinh thần “RAG là kỹ năng được săn đón nhiều nhất hiện tại cho AI Engineer” trong nguồn.

## ⚠️ 4 lỗi thường gặp (mở rộng)

### 1. Dùng LangChain từ đầu → không hiểu pipeline, không debug được

**Tại sao lỗi này xảy ra:** LangChain/LlamaIndex gói gọn load-split-embed-retrieve-generate trong vài dòng code — tiện nhưng che khuất toàn bộ dữ liệu chảy giữa các bước. Khi retrieval kém, bạn không biết do chunk sai, do embedding lệch, hay do prompt — vì mọi thứ đều là "magic" của framework. Thêm nữa, API framework đổi nhanh giữa các phiên bản; kiến thức "cách dùng LangChain" cũ đi rất nhanh, còn kiến thức "pipeline RAG hoạt động thế nào" thì không.

**Ví dụ cụ thể:** bạn dùng `RetrievalQA.from_chain_type(...)` và câu trả lời sai. Bug nằm ở đâu? Ở `RecursiveCharacterTextSplitter` cắt trúng câu? Ở embedding model không hỗ trợ tiếng Việt? Ở prompt không ép citation? Cả ba đều nằm sau một dòng code — bạn phải đọc source framework để biết, thay vì đặt breakpoint vào đúng chỗ nếu tự viết.

**Cách tránh:** **Viết tay trước, framework sau.** V1 project (file 07) cấm LangChain/LlamaIndex là vì thế — vài trăm dòng code thuần dạy bạn nhiều hơn vài tuần dùng framework. Khi đã hiểu pipeline, dùng framework là lựa chọn năng suất, không phải chỗ dựa.

### 2. Quên eval → sửa pipeline trong mù, không biết cải tiến có thật sự tốt hơn

**Tại sao lỗi này xảy ra:** thay chunk size 512 → 256 rồi hỏi thử 3 câu thấy "có vẻ tốt hơn" — nhưng 3 câu đó dễ hơn/thiên vị, và cảm giác "tốt hơn" thường là placebo. Không có số liệu trước/sau trên **cùng bộ câu hỏi**, mọi kết luận chỉ là cảm tính; tệ hơn, bạn có thể đang làm mọi thứ xấu đi mà không biết.

**Cách tránh:** xây eval dataset 30+ câu hỏi + ground truth ngay từ đầu (không đợi xong pipeline), rồi đo bằng Ragas **mỗi khi đổi một biến** (file 06). Giữ bảng so sánh naive vs +rerank vs +hybrid — đó vừa là bằng chứng cải tiến, vừa là DoD của project (file 07).

### 3. Chunk size để mặc định (1024) → retrieval kém

**Tại sao lỗi này xảy ra:** 1024 token là con số "mặc định cho tiện" — nó pha loãng nhiều chủ đề vào một chunk, khiến embedding trung bình hóa ngữ nghĩa và không "bắt" được câu hỏi cụ thể; đồng thời context dài pha loãng attention của LLM. Nhưng cũng không có con số thần thánh nào hợp mọi dữ liệu — chunk 256 lại dễ mất ngữ cảnh với tài liệu dày đặc thông tin.

**Cách tránh:** **thử 256/512/1024 và đo** sự khác biệt bằng eval (đo context precision/recall — file 06). Kết hợp overlap 10–20% (ví dụ chunk 512 overlap 64 — file 03) và chunk theo heading khi tài liệu có cấu trúc. Quy tắc: sửa chunk size là **một biến trong vòng lặp eval**, không phải quyết định một lần.

### 4. Chỉ tối ưu generation prompt, bỏ qua retrieval → 90% vấn đề RAG nằm ở retrieval

**Tại sao lỗi này xảy ra:** prompt là thứ thấy ngay trước mắt nên ai cũng muốn sửa ("viết lại prompt hay hơn!"), còn retrieval thì ẩn sau vài dòng truy vấn. Nhưng nếu context đưa vào prompt đã sai/thiếu (retrieval kém), prompt hay đến mấy cũng chỉ giúp LLM diễn đạt mượt hơn cho **thông tin sai** — "garbage in, garbage out". Con số 90% là kinh nghiệm cộng đồng: phần lớn câu trả lời tệ truy ngược về chunk không liên quan hoặc thiếu.

**Cách tránh:** khi câu trả lời kém, **chẩn đoán bằng metric trước** (file 06): context recall thấp → sửa retrieval (multi-query, hybrid, chunking); faithfulness thấp với context đủ → mới sửa prompt/generation (file 05). Ưu tiên đo retrieval (context precision/recall) trước khi đụng tới prompt.

**Ma trận nhanh:** context recall thấp → sửa chunking/multi-query/hybrid; context precision thấp → thêm reranker; cả hai ổn mà faithfulness vẫn thấp → lúc đó mới sửa prompt. Thứ tự này tiết kiệm hàng tuần công sức vô ích.

## Liên kết

- [01 — Embedding & Vector Search](01-embedding-and-vector-search.md) — dùng MTEB Leaderboard để chọn model
- [03 — Ingestion & Chunking](03-ingestion-parsing-chunking.md) — lỗi chunk size mặc định
- [06 — Evaluation với Ragas](06-evaluation-ragas.md) — giải pháp cho lỗi quên eval
- [07 — Project Docs-QA](07-project-docs-qa.md) — project áp dụng tất cả điều trên
