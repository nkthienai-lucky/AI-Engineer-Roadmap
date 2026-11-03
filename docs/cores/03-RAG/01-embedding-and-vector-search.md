# Embedding & Vector Search (Tuần 8)

> **Mục tiêu:** Hiểu embedding model hoạt động thế nào, biết cách chọn model theo MTEB / đa ngôn ngữ / tiếng Việt, nắm công thức và trường hợp dùng của cosine / dot product / L2, và có intuition về ANN (HNSW, IVF) — tại sao tìm "gần đúng" mà nhanh hơn nhiều so với tìm chính xác.
> Nguồn mở rộng từ: [03-RAG.md](../../../03-RAG.md)

---

## Embedding model hoạt động thế nào

Embedding model biến một đoạn văn bản (hoặc ảnh, code…) thành một **vector số dày đặc** (dense vector) — ví dụ 384, 768, 1024 hoặc 1536 chiều. Vector này "mã hóa" ngữ nghĩa của văn bản: hai câu có nghĩa giống nhau sẽ cho hai vector nằm gần nhau trong không gian vector, câu có nghĩa khác nhau sẽ nằm xa nhau.

Cách hoạt động ở mức đơn giản:

1. Văn bản được tách thành token, đi qua một mô hình ngôn ngữ (thường là encoder kiểu BERT/Sentence-Transformer) huấn luyện sẵn.
2. Mô hình xuất ra một vector duy nhất cho cả câu/đoạn (thường là pooling lớp cuối — lấy trung bình hoặc token `[CLS]`).
3. Trong quá trình huấn luyện, mô hình được tối ưu theo kiểu **contrastive**: cặp câu tương đồng thì kéo vector lại gần, cặp câu không liên quan thì đẩy ra xa. Vì vậy khoảng cách trong không gian vector phản ánh độ tương đồng ngữ nghĩa.

Ví dụ dùng embedding API (OpenAI-compatible hoặc model local qua `sentence-transformers`):

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("intfloat/multilingual-e5-large")
vec = model.encode("RAG là gì?")
print(vec.shape)  # (1024,) — 1 vector 1024 chiều cho 1 câu
```

Lưu ý quan trọng: vector chỉ "có nghĩa" khi so sánh với các vector **cùng model, cùng chiều**. Không trộn vector từ hai model khác nhau trong một index.

## Chọn embedding model theo tiêu chí gì

- **MTEB (Massive Text Embedding Benchmark):** bảng xếp hạng chuẩn để so sánh hàng trăm model trên nhiều tác vụ (classification, retrieval, clustering, semantic similarity…). Khi chọn model, nhìn điểm **Retrieval** trên MTEB là quan trọng nhất với use-case RAG. Xem bảng xếp hạng chính thức: [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard).
- **Đa ngôn ngữ / tiếng Việt:** nếu tài liệu của bạn có tiếng Việt, **bắt buộc** chọn model đa ngôn ngữ có hỗ trợ tốt tiếng Việt (ví dụ họ `multilingual-e5`, `bge-m3`, hoặc model của Cohere/OpenAI đa ngôn ngữ). Model chỉ train trên tiếng Anh (ví dụ `all-MiniLM-L6-v2`) sẽ cho embedding tiếng Việt rất kém — hai câu tiếng Việt khác nghĩa vẫn có thể ra vector gần nhau.
- **Chiều vector & chi phí:** vector càng dài thì chi phí lưu trữ và tìm kiếm càng cao (1 triệu vector × 1024 chiều × 4 byte ≈ 4 GB RAM). Model nhỏ (384 chiều) chạy nhanh, model lớn (1024+) chính xác hơn nhưng đắt hơn. Đo trên **bộ dữ liệu của chính bạn** thay vì chỉ tin leaderboard.
- **Nhúng cục bộ hay API:** model local (`sentence-transformers`) miễn phí, dữ liệu không rời máy, cần GPU/CPU đủ mạnh. Embedding API (OpenAI, Cohere…) chất lượng ổn định, không phải lo hạ tầng, nhưng tốn chi phí theo token và dữ liệu đi ra ngoài.

## Vector similarity: cosine, dot product, L2

Cho hai vector $u, v \in \mathbb{R}^n$:

### Cosine similarity

$$\text{cos}(u, v) = \frac{u \cdot v}{\|u\|\,\|v\|} = \frac{\sum_i u_i v_i}{\sqrt{\sum_i u_i^2}\,\sqrt{\sum_i v_i^2}}$$

Chỉ đo **hướng** của vector, không đo độ dài. Giá trị trong khoảng $[-1, 1]$ (1 = cùng hướng hoàn toàn). Đây là metric **mặc định và phổ biến nhất** cho text embedding, vì hầu hết model được huấn luyện để chuẩn hóa vector về cùng độ dài — khi đó cosine và dot product cho thứ tự xếp hạng như nhau.

### Dot product (inner product)

$$u \cdot v = \sum_i u_i v_i$$

Đo cả hướng lẫn **độ lớn** (magnitude). Nếu vector đã được chuẩn hóa (unit norm) thì dot product = cosine. Dùng dot product khi model được huấn luyện với norm mang thông tin (ví dụ một số model E5, BGE) — vector "tự tin" hơn có norm lớn hơn.

### L2 (Euclidean distance)

$$\|u - v\|_2 = \sqrt{\sum_i (u_i - v_i)^2}$$

Đo **khoảng cách hình học** thẳng giữa hai điểm. Càng nhỏ càng giống. Với vector chuẩn hóa, L2 và cosine cho cùng thứ tự xếp hạng ($\|u-v\|^2 = 2 - 2\cos(u,v)$ với vector đơn vị).

### Khi nào dùng cái nào

| Metric | Dùng khi |
|---|---|
| Cosine | Mặc định cho text embedding; không quan tâm độ dài vector |
| Dot product | Model chuẩn hóa sẵn hoặc norm chứa thông tin độ tin cậy |
| L2 | Bài toán clustering/hình học, hoặc framework chỉ hỗ trợ Euclidean |

**Quy tắc thực hành:** xem documentation của embedding model — họ ghi rõ metric huấn luyện. Dùng đúng metric đó khi index và khi truy vấn, nếu không kết quả sẽ lệch.

## ANN (Approximate Nearest Neighbor) — tìm gần đúng mà nhanh

Bài toán: với vector câu hỏi, tìm k vector gần nhất trong hàng triệu vector đã lưu. Quét toàn bộ (brute-force, exact search) độ phức tạp $O(N \cdot d)$ — với 10 triệu vector, mỗi truy vấn phải tính 10 triệu phép so sánh. Quá chậm cho production.

ANN chấp nhận **đánh đổi một chút độ chính xác để lấy tốc độ vượt bậc** (thường recall 95–99% mà nhanh gấp 10–1000 lần). Intuition: thay vì so sánh với tất cả, ta dựng một cấu trúc dữ liệu cho phép **đi thẳng vào vùng có khả năng chứa vector gần nhất**.

### HNSW (Hierarchical Navigable Small World)

Intuition: giống cách con người tìm đường trên bản đồ nhiều tầng. HNSW dựng **nhiều tầng đồ thị**:

- Tầng trên cùng rất thưa — mỗi node chỉ nối với vài node xa nhau → nhảy những bước dài để "định vị" vùng.
- Càng xuống dưới, đồ thị càng dày → tinh chỉnh dần đến điểm gần nhất.
- Truy vấn: bắt đầu ở tầng thưa nhất, greedily đi tới node gần hơn, rồi hạ tầng, lặp lại cho tới tầng đáy — nơi có đầy đủ mọi node.

Kết quả: thay vì so sánh với $N$ vector, ta chỉ duyệt vài trăm node → cực nhanh. HNSW là thuật toán ANN mặc định phổ biến nhất hiện nay (pgvector dùng `vector_cosine_ops` HNSW, Qdrant dùng HNSW nội bộ).

### IVF (Inverted File Index)

Intuition: **chia để trị**. Dùng clustering (k-means) chia không gian vector thành $n_{list}$ cụm (Voronoi cells), mỗi cụm lưu danh sách các vector thuộc nó ("inverted file"). Truy vấn:

1. Tìm $n_{probe}$ cụm có tâm gần vector truy vấn nhất (chỉ so sánh với $n_{list}$ tâm — rất ít phép tính).
2. Quét (brute-force) **trong nội bộ** các cụm đó thay vì toàn bộ dữ liệu.

Tăng $n_{probe}$ → chính xác hơn nhưng chậm hơn. IVF hay dùng trong FAISS; pgvector cũng hỗ trợ `ivfflat`.

### So sánh nhanh

| | HNSW | IVF |
|---|---|---|
| Độ chính xác | Rất cao (graph-based) | Phụ thuộc số cụm / n_probe |
| Bộ nhớ | Tốn hơn (lưu đồ thị nhiều tầng) | Nhẹ hơn |
| Xây index | Chậm hơn (dựng graph) | Nhanh hơn (train k-means) |
| Intuition | Nhảy nhiều tầng để thu hẹp | Chia cụm rồi chỉ tìm trong vài cụm |

**Điểm mấu chốt:** ANN đổi "chính xác tuyệt đối" lấy "chính xác đủ tốt + latency thấp". Với RAG, recall 95% ở top-k là hoàn toàn chấp nhận được — và ta còn có reranking (file 04) để bù lại phần sai lệch này.

## Liên kết

- [02 — Vector DB: pgvector & Qdrant](02-vector-db-pgvector-qdrant.md) — nơi lưu và truy vấn các vector này
- [03 — Ingestion & Chunking](03-ingestion-parsing-chunking.md) — tạo ra các đoạn văn bản cần embedding
- [08 — Tài nguyên & Lỗi thường gặp](08-resources-and-common-pitfalls.md) — MTEB Leaderboard, tài liệu học thêm
