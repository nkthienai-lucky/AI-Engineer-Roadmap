# Project GĐ3 — "Docs-QA có trích dẫn + Eval"

> **Mục tiêu:** Spec đầy đủ cho project giai đoạn 3 — Docs-QA app chat với PDF/website, **có trích dẫn nguồn** và **đánh giá chất lượng bằng số liệu**. Ba phiên bản tăng dần (V1 viết tay → V2 reranker + hybrid → V3 eval Ragas) + UI web + Definition of Done + checklist nghiệm thu.
> Nguồn mở rộng từ: [03-RAG](README.md)

---

## Tổng quan

**Đầu ra:** Docs-QA app (chat với PDF/website, trích dẫn nguồn) + báo cáo eval. Đây là project "hái ra tiền" nhất của roadmap — RAG là kỹ năng được săn đón nhiều nhất hiện tại cho AI Engineer, và project này demo đúng hai thứ nhà tuyển dụng hỏi: **pipeline tự build được** và **đo lường được chất lượng**.

**Yêu cầu gốc (giữ nguyên từ nguồn):**

1. **Phiên bản 1 (tuần 9):** RAG pipeline **viết tay, KHÔNG dùng LangChain/LlamaIndex** — chỉ dùng embedding API + pgvector/Qdrant + LLM API
2. **Phiên bản 2 (tuần 10):** Thêm reranker + hybrid search
3. **Phiên bản 3 (tuần 11):** Eval với Ragas trên 30+ câu hỏi test; in bảng so sánh trước/sau cải tiến
4. UI web (Gradio/Streamlit): upload PDF → hỏi → trả lời kèm nguồn (trang X, đoạn Y)

## V1 — RAG pipeline viết tay (tuần 9)

**Mục tiêu:** tự tay nối các mắt xích, không framework — để hiểu **đúng** pipeline thay vì magic. Chỉ dùng: embedding API (hoặc `sentence-transformers`) + pgvector/Qdrant + LLM API.

Phạm vi (theo [03 — Ingestion](03-ingestion-parsing-chunking.md)):

1. **Ingest:** upload PDF → parse (`Docling`/`Unstructured`) → chunk (thử 256/512/1024, có overlap 64) → embed → store vào pgvector/Qdrant kèm metadata (`doc_id`, `page`, `section`).
2. **Retrieve:** embed câu hỏi → top-k cosine similarity (chưa rerank).
3. **Generate:** ghép context vào prompt chống hallucination ([file 05](05-generation-citation-anti-hallucination.md)) → LLM trả lời kèm citation `[số]` → UI hiển thị "trang X, đoạn Y", click ra đoạn gốc.

Viết tay = bạn tự gọi embedding API, tự viết SQL/query Qdrant, tự ghép prompt — tổng mã chỉ vài trăm dòng, nhưng bạn hiểu **từng dòng**.

## V2 — Reranker + hybrid search (tuần 10)

Giữ nguyên V1, thêm hai cải thiện (theo [file 04](04-retrieval-reranking-hybrid.md)):

- **Reranker cross-encoder** (`bge-reranker`): retrieval top-50 → rerank → top-5 vào prompt.
- **Hybrid search:** BM25 + vector, gộp bằng **RRF fusion**.
- Bonus (nếu có thời gian): query rewriting / multi-query cho câu hỏi mơ hồ.

Mọi thay đổi phải **đo được** — chạy lại eval set để so với V1 (đầy đủ ở V3).

## V3 — Eval Ragas 30+ câu hỏi (tuần 11)

- Xây eval dataset **30+ câu hỏi** + ground truth (theo tiêu chuẩn [file 06](06-evaluation-ragas.md)): đa dạng fact/suy luận/nhiều chunk/câu ngoài tầm/từ khóa hiếm.
- Chạy **Ragas** (bắt buộc) trên cả ba cấu hình: naive (V1) → +rerank (V2a) → +hybrid (V2b).
- **In bảng so sánh trước/sau cải tiến** bằng số liệu — ví dụ:

| Cấu hình | Faithfulness | Answer relevancy | Context precision | Context recall |
|---|---|---|---|---|
| Naive | 0.72 | 0.80 | 0.61 | 0.58 |
| + Rerank | 0.78 | 0.83 | 0.79 | 0.63 |
| + Hybrid | 0.84 | 0.86 | 0.81 | 0.74 |

*(Bảng trên là ví dụ tự nghĩ để minh họa format — con số thực tế lấy từ eval run của bạn.)*

## UI web (Gradio/Streamlit)

Luồng người dùng: **upload PDF → hỏi → trả lời kèm nguồn (trang X, đoạn Y)**. Yêu cầu cụ thể:

- Ô upload PDF + danh sách tài liệu đã index.
- Ô chat hỏi đáp; câu trả lời hiển thị citation `[số]`.
- Danh sách nguồn bên dưới/trong popup: `handbook.pdf — trang 12, đoạn "..."` — **click ra được đoạn gốc** (cuộn tới hoặc mở modal chứa text chunk).

## Tiêu chí hoàn thành (DoD)

- [ ] Pipeline tay chạy được end-to-end (không framework)
- [ ] Trả lời có trích dẫn nguồn, click ra được đoạn gốc
- [ ] Bảng eval Ragas: faithfulness > 0.8, context recall > 0.7
- [ ] Có bảng so sánh "naive vs +rerank vs +hybrid" bằng số liệu
- [ ] Deploy lên HF Spaces hoặc Oracle VPS

## Checklist nghiệm thu

- [ ] **V1:** nạp được PDF → chat trả lời đúng, có citation trang/đoạn; mã **không** import LangChain/LlamaIndex
- [ ] **V1:** chunk metadata (`doc_id`, `page`) lưu đầy đủ, filter theo tài liệu hoạt động
- [ ] **V2:** reranker chạy trên top-50 → top-5; kết quả khác (tốt hơn) V1 trên eval set
- [ ] **V2:** hybrid BM25 + vector + RRF hoạt động; câu hỏi có mã sản phẩm/từ hiếm tìm được
- [ ] **V3:** eval dataset 30+ câu có ground truth, đa dạng kiểu câu
- [ ] **V3:** chạy Ragas in đủ 4 metric; bảng so sánh naive vs +rerank vs +hybrid có số liệu thật
- [ ] **V3:** đạt DoD faithfulness > 0.8 và context recall > 0.7
- [ ] **UI:** upload → hỏi → trả lời kèm nguồn, click ra được đoạn gốc
- [ ] **Deploy:** app chạy trên HF Spaces hoặc Oracle VPS, người khác truy cập được

## Liên kết

- [03 — Ingestion & Chunking](03-ingestion-parsing-chunking.md) — phần việc của V1 (tuần 9)
- [04 — Retrieval & Hybrid Search](04-retrieval-reranking-hybrid.md) — phần việc của V2 (tuần 10)
- [05 — Generation & Chống hallucination](05-generation-citation-anti-hallucination.md) — prompt + citation cho UI
- [06 — Evaluation với Ragas](06-evaluation-ragas.md) — phần việc của V3 (tuần 11)
- [08 — Tài nguyên & Lỗi thường gặp](08-resources-and-common-pitfalls.md) — tránh các bẫy khi làm project
