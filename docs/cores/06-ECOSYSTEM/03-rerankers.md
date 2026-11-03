# 03 — Rerankers: bảng tra & khi nào cần rerank

> **Mục tiêu**: hiểu cross-encoder khác gì retrieval bằng embedding, khi nào đáng thêm bước rerank, và chọn tool rerank phù hợp (không học thuộc).
>
> Nguồn mở rộng từ: [06-ECOSYSTEM](README.md)

---

## Bảng tra nhanh

| Tool | Loại | Ghi chú |
|---|---|---|
| **bge-reranker-base/v2-m3** | Cross-encoder, open source | Chạy CPU OK — khuyến nghị |
| FlashRank | Nhẹ, CPU-first | Nhanh, chất lượng khá |
| Cohere Rerank | API trả phí | Chất lượng cao, tính theo call |

---

## Cross-encoder khác gì embedding retrieval?

- **Embedding retrieval (bi-encoder)**: query và mỗi document được encode **riêng biệt** thành vector, so sánh bằng cosine similarity. Nhanh (document tính trước, chỉ tính vector query lúc truy vấn) → hợp để tìm trong hàng triệu docs. Nhưng model không thấy query và document cùng lúc nên chỉ ước lượng "độ liên quan".
- **Cross-encoder**: đưa **cặp (query, document)** vào model cùng lúc, output ra **một điểm relevance** cho đúng cặp đó. Chính xác hơn hẳn vì model chú ý trực tiếp mối quan hệ query–doc. Trade-off: phải chạy model cho **từng cặp** → chậm, không tính trước được → chỉ dùng cho vài chục ứng viên, không dùng để tìm trong toàn bộ corpus.
- Quy trình chuẩn: **retrieval (bi-encoder/vector DB) lấy top-k ~20–100 → rerank (cross-encoder) chấm lại → lấy top 5 đưa vào LLM.** Hai bước bổ trợ nhau, không thay thế nhau.

## Khi nào cần rerank?

- **Cần khi**: chất lượng context quyết định chất lượng câu trả lời (RAG production), corpus có nhiều document na ná nhau (tài liệu kỹ năng, luật, FAQ), truy vấn dài/phức tạp, hoặc bạn thấy context retrieve về đúng chủ đề nhưng sai chi tiết.
- **Không cần khi**: corpus nhỏ, câu trả lời vẫn tốt với top-k từ vector DB, hoặc hệ thống cực nhạy latency (mỗi cặp rerank thêm ~10–100ms CPU).
- Chi phí rerank thấp so với lợi ích: chạy CPU là được, chỉ trên top-k nên nhẹ.

## Luồng rerank cụ thể (hình dung)

1. User query → embedding (bi-encoder, [02-embedding-models.md](02-embedding-models.md)).
2. Vector DB ([01-vector-db.md](01-vector-db.md)) trả **top-k ~20–100** ứng viên (k lấy quá lớn để rerank không chậm, quá nhỏ thì dễ bỏ sót).
3. Cross-encoder chấm điểm relevance cho **từng cặp (query, doc)** → sắp xếp lại.
4. Lấy **top 5** (hoặc số chunk phù hợp context window) đưa vào prompt cho LLM sinh câu trả lời.
- Số liệu cần nhớ khi thiết kế: rerank thêm một lần inference cho mỗi cặp — với k=50 trên CPU là vài chục ms đến vài trăm ms tùy model; đổi lại context chính xác hơn → câu trả lời ít "đúng chủ đề nhưng sai chi tiết" hơn.

## Chọn tool

- **bge-reranker-base/v2-m3** (cross-encoder, open source): khuyến nghị mặc định — chạy CPU OK, v2-m3 đa ngôn ngữ mạnh (hợp tiếng Việt). Dùng được local, không phụ thuộc API. `base` nhẹ hơn cho máy yếu; `v2-m3` tốt hơn khi corpus đa ngôn ngữ.
- **FlashRank**: thư viện nhẹ, thiết kế CPU-first — nhanh, chất lượng khá. Hợp khi cần rerank nhiều query, ngại nặng.
- **Cohere Rerank**: API trả phí, chất lượng cao, tính tiền theo call. Hợp khi không muốn host model và sẵn sàng trả.
- Tiêu chí chọn nhanh: ưu tiên local + CPU (bge-reranker) → cần tốc độ tối đa (FlashRank) → muốn chất lượng cao nhất không lo host (Cohere).

---

## Liên kết

- [04-rag-pipeline-tools.md](04-rag-pipeline-tools.md) — rerank nằm giữa retrieval và generation trong pipeline RAG (xem thêm [03-RAG](../03-RAG/README.md)).
- [01-vector-db.md](01-vector-db.md) — bước retrieval top-k trước khi rerank.
- [02-embedding-models.md](02-embedding-models.md) — bi-encoder dùng cho bước retrieval đối chiếu với cross-encoder ở đây.
