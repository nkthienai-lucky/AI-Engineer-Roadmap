# Retrieval, Reranking & Hybrid Search (Tuần 10)

> **Mục tiêu:** Hiểu Naive RAG và các cải thiện (query rewriting, HyDE, multi-query), hiểu vì sao cần reranking cross-encoder (`bge-reranker`) khi top-k retrieval ≠ top-k đúng (bi-encoder vs cross-encoder), và biết kết hợp BM25 + vector trong hybrid search với RRF fusion.
> Nguồn mở rộng từ: [03-RAG](README.md)

---

## Naive RAG và các cải thiện

**Naive RAG** = embed câu hỏi → tìm top-k vector gần nhất → ném vào prompt → LLM trả lời. Nó hoạt động, nhưng có nhiều điểm yếu: câu hỏi người dùng viết mơ hồ/lỗi chính tả, khác "phong cách" với văn bản tài liệu, và một lần truy vấn duy nhất có thể bỏ sót tài liệu liên quan. Các kỹ thuật sau cải thiện **chất lượng truy vấn và tập context** trước khi sinh câu trả lời.

### Query rewriting

Đưa câu hỏi (có thể ngắn, khẩu ngữ, chứa đại từ) qua một LLM call để viết lại thành **truy vấn rõ ràng, đầy đủ ngữ cảnh, đúng ngôn ngữ/từ vựng của tài liệu**. Ví dụ: *"Nó bảo hành bao lâu?"* → *"Thời hạn bảo hành sản phẩm là bao lâu?"*.

**Khi nào dùng:** mọi lúc — chi phí gần như bằng 0 (một LLM call nhỏ) mà cải thiện retrieval thấy rõ, đặc biệt với chat nhiều lượt (cần mang ngữ cảnh lượt trước vào câu hỏi hiện tại).

### HyDE (Hypothetical Document Embeddings)

Ý tưởng đảo ngược: thay vì embed **câu hỏi**, hãy để LLM **viết một câu trả lời giả định** (hypothetical document) cho câu hỏi đó, rồi embed **câu trả lời giả** để tìm kiếm. Vì embedding trả lời giả gần với embedding tài liệu thật (cùng "phong cách" diễn giải) hơn là embedding câu hỏi, kết quả retrieval tốt hơn đáng kể.

**Khi nào dùng:** câu hỏi khó, trừu tượng, khác biệt lớn giữa cách hỏi và cách viết trong tài liệu. Không dùng khi câu hỏi đã rõ ràng (tốn LLM call vô ích) hoặc model sinh text giả yếu.

### Multi-query

Cho LLM sinh **nhiều biến thể câu hỏi** (đồng nghĩa, khác góc nhìn, chi tiết hơn), truy vấn cả biến thể, rồi **gộp kết quả** (union + dedupe theo độ tương đồng). Một câu hỏi duy nhất chỉ "bắn" được một hướng trong không gian vector; multi-query bắn nhiều hướng → recall cao hơn.

**Khi nào dùng:** câu hỏi phức hợp nhiều khía cạnh, hoặc khi eval cho thấy **context recall thấp** (tài liệu đúng không được lọt vào top-k). Chi phí: nhiều embedding search hơn nhưng chỉ 1 LLM call sinh query.

## Reranking: cross-encoder (`bge-reranker`)

### Tại sao top-k retrieval ≠ top-k đúng

Vector search trả về 20–50 ứng viên theo độ tương đồng **thô** — nhưng thứ tự đó thường sai ở top: ứng viên đứng #1 có thể chỉ liên quan bề mặt (dùng chung từ khóa), còn tài liệu trả lời đúng đứng #5. Nếu chỉ lấy top-5 đưa vào LLM, ta có thể **bỏ lỡ đúng mảnh ghép cần thiết**. Reranking sinh ra để sắp xếp lại 20–50 ứng viên đó thành thứ tự **chính xác nhất** rồi mới lấy top-k cuối cùng.

### Bi-encoder vs cross-encoder

| | Bi-encoder (dùng khi index) | Cross-encoder (dùng khi rerank) |
|---|---|---|
| Cách hoạt động | Encode câu hỏi và chunk **riêng biệt**, so sánh hai vector | Đưa **(câu hỏi, chunk) cùng lúc** vào model, model "đọc" cả hai cùng nhau |
| Tốc độ | Nhanh (chunk được embed trước, chỉ embed 1 lần) | Chậm (chấm từng cặp — nhưng chỉ trên vài chục ứng viên) |
| Độ chính xác | Khá (không thấy sự tương tác giữa hai văn bản) | Rất cao (attention bắt được mối liên hệ tinh tế) |
| Vai trò trong pipeline | **Retrieval** — lọc triệu chunk xuống vài chục | **Rerank** — sắp xếp lại vài chục thành top-k đúng |

Intuition: bi-encoder như hai người mô tả riêng từng bức tranh rồi so khớp lời mô tả — nhanh nhưng thô. Cross-encoder như một giám khảo đặt câu hỏi và bức tranh **cạnh nhau** rồi chấm — kỹ hơn nhưng mỗi lần chỉ chấm được một cặp.

### bge-reranker

Họ model reranker của BAAI, miễn phí, chạy local, chất lượng cao (ví dụ `BAAI/bge-reranker-v2-m3` hỗ trợ đa ngôn ngữ — quan trọng với tiếng Việt). Cách dùng:

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("BAAI/bge-reranker-v2-m3")
pairs = [(query, chunk) for chunk in retrieved_chunks]   # 20–50 cặp
scores = reranker.predict(pairs)
ranked = [c for _, c in sorted(zip(scores, retrieved_chunks), reverse=True)]
top_k = ranked[:5]   # 5 chunk "đúng" nhất đưa vào prompt
```

**Cấu trúc chuẩn của pipeline cải thiện:** bi-encoder retrieval (top-50) → cross-encoder rerank → lấy top-5 cho LLM.

## Hybrid search: BM25 + vector (RRF fusion)

Vector search giỏi **ngữ nghĩa** ("bảo hành bao lâu" khớp "thời hạn bảo hành") nhưng **dễ bỏ sót từ khóa hiếm** (mã sản phẩm "TX-9000", tên riêng, số hiệu). BM25 (search từ khóa cổ điển) thì ngược lại: khớp chính xác từ khóa nhưng không hiểu đồng nghĩa.

**Hybrid search** chạy **cả hai** rồi gộp kết quả:

1. **BM25:** truy vấn từ khóa trên text chunks (Postgres có `tsvector`/`ts_rank`; Qdrant build sẵn sparse vector BM25).
2. **Vector:** truy vấn embedding như bình thường.
3. **Gộp bằng RRF (Reciprocal Rank Fusion):** không so sánh điểm của hai hệ thống (thang điểm khác nhau, không so được), mà so sánh **thứ hạng**:

$$\text{RRF}(d) = \sum_{r \in R} \frac{1}{k + \text{rank}_r(d)}$$

với $k \approx 60$ (hằng số chuẩn), $R$ là tập các bảng xếp hạng. Tài liệu có mặt ở cả hai hệ thống, hoặc đứng hạng cao trong một hệ thống, sẽ được đẩy lên trên. RRF đơn giản, không cần tune trọng số, và hoạt động tốt bất chấp thang điểm khác nhau.

**Khi nào dùng:** tài liệu có nhiều thuật ngữ chuyên môn/mã số/tên riêng (y tế, pháp lý, kỹ thuật) — tức là khi eval cho thấy câu hỏi về từ khóa hiếm bị retrieval bỏ sót.

## Liên kết

- [03 — Ingestion & Chunking](03-ingestion-parsing-chunking.md) — retrieval tốt xấu phụ thuộc chất lượng chunk
- [05 — Generation & Chống hallucination](05-generation-citation-anti-hallucination.md) — dùng top-k đã rerank để sinh câu trả lời
- [06 — Evaluation với Ragas](06-evaluation-ragas.md) — đo context precision/recall để biết có cần rerank/hybrid không
- [07 — Project Docs-QA](07-project-docs-qa.md) — V2 thêm reranker + hybrid search
