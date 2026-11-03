# 01 — Vector DB: bảng tra & cách chọn

> **Mục tiêu**: tra nhanh khi cần chọn vector database cho RAG — hiểu từng "loại" DB là gì, cột "tự host free"/"cloud free tier" nghĩa gì, và khi nào dùng công cụ nào (không học thuộc).
>
> Nguồn mở rộng từ: [06-ECOSYSTEM](README.md)

---

## Bảng tra nhanh

| Vector DB | Loại | Tự host free | Cloud free tier | Hợp với |
|---|---|---|---|---|
| **pgvector** | Extension Postgres | ✅ (cùng Postgres) | Supabase / Neon free | **Mặc định nên dùng** — không thêm service, dễ deploy |
| **Qdrant** | Purpose-built (Rust) | ✅ Docker, nhẹ | 1GB cluster, 4 vCPU | Performance cao, filter mạnh |
| **Chroma** | Embedded, nhẹ | ✅ 1 dòng cài | Có managed mới | Học tập, prototype notebook |
| **Weaviate** | Purpose-built | ✅ Docker | Sandbox 14 ngày | Hybrid search (text + vector) |
| **Milvus / Zilliz** | Scale lớn | ✅ (nặng) | Zilliz: 5GB free | Hàng triệu vectors |
| **FAISS** | Library (không server) | ✅ | ❌ | Nghiên cứu, tìm hiểu thuật toán |
| **Pinecone** | Managed SaaS | ❌ | Free tier có hạn | Không muốn tự quản lý |

---

## Cột "Loại" nghĩa là gì?

- **Extension Postgres** (pgvector): không phải phần mềm riêng — chỉ là một extension cài vào Postgres bạn đang có. Vector được lưu thành kiểu dữ liệu trong bảng quen thuộc, truy vấn chung với WHERE/JOIN của SQL. Lợi: không thêm service nào để vận hành, backup/transaction dùng chung Postgres. Trade-off: hiệu năng tìm kiếm vector không bằng DB chuyên dụng khi dữ liệu lớn, nhưng với hầu hết dự án vừa là đủ.
- **Purpose-built** (Qdrant, Weaviate): DB thiết kế riêng cho vector search từ đầu. Có sẵn HNSW index, metadata filter mạnh, hybrid search. Trade-off: phải chạy thêm một service (thường qua Docker) — nhưng Qdrant (Rust) rất nhẹ, Weaviate mạnh về hybrid text + vector.
- **Embedded, nhẹ** (Chroma): chạy ngay trong process Python của bạn, không cần server. Cài 1 dòng, dùng như thư viện. Trade-off: không hợp nhiều user/production thật, nhưng cực tiện cho notebook và prototype.
- **Library (không server)** (FAISS): thư viện C++/Python của Meta — chỉ là tập hàm build index + tìm kiếm, không có storage server, không auth, không API. Bạn tự lo mọi thứ xung quanh. Hợp nghiên cứu/hiểu thuật toán, không hợp "deploy rồi quên".
- **Managed SaaS** (Pinecone): nhà cung cấp lo toàn bộ (hosting, scaling, backup), bạn chỉ gọi API. Trade-off: tiện nhất nhưng không tự host được (cột "tự host free" = ❌), phụ thuộc nhà cung cấp, chi phí tăng theo usage.
- **Scale lớn** (Milvus/Zilliz): kiến trúc phân mảnh (sharding) cho hàng triệu–tỷ vectors. Self-host Milvus "nặng" (nhiều thành phần: etcd, MinIO, Pulsar) — Zilliz là bản managed cloud.

## Cột "Tự host free" vs "Cloud free tier"

- **Tự host free**: cài trên máy của bạn (VPS, laptop) không tốn phí license. "✅ (cùng Postgres)" nghĩa là pgvector gần như free vì Postgres đã có sẵn. "✅ Docker, nhẹ" nghĩa là chạy container xong là dùng. "✅ (nặng)" cảnh báo cần tài nguyên kha khá (Milvus).
- **Cloud free tier**: bản miễn phí của dịch vụ cloud do vendor cung cấp — hữu ích để thử nhanh không cần cài, nhưng luôn có giới hạn (dung lượng, vCPU, thời gian). Ví dụ: Qdrant free 1GB cluster/4 vCPU, Weaviate Sandbox chỉ 14 ngày (hết hạn thì mất data hoặc phải nâng cấp), Zilliz 5GB free, Pinecone free tier "có hạn" (bị giới hạn index/queries). FAISS không có cloud vì nó chỉ là library.

---

## Chọn nhanh

- **Học → `Chroma`/`FAISS`**: mục tiêu là hiểu vector search hoạt động thế nào, không cần vận hành. Chroma cài 1 dòng, chạy trong notebook; FAISS cho ai muốn đọc code index HNSW/IVF.
- **Deploy Oracle VPS → `pgvector`**: VPS free của Oracle thường đã chạy Postgres hoặc sắp xếp được Postgres — thêm pgvector là có vector DB, không thêm service, không thêm RAM hao phí, dễ backup. Đây là "mặc định nên dùng" cho dự án thật ở quy mô vừa.
- **Production lớn → `Qdrant`/`Weaviate`**: khi cần performance cao, metadata filter phức tạp (Qdrant) hoặc hybrid search text + vector (Weaviate), và đã có đội/dịch vụ chạy Docker ổn định.

---

## Liên kết chi tiết (trong nguồn)

- So sánh chi tiết: [buttondown.com/ultradune/archive/eval-002-vector-databases-in-2026](https://buttondown.com/ultradune/archive/eval-002-vector-databases-in-2026-qdrant-vs/) · [aitoolguide.ai/blog/best-vector-databases-for-rag-2026](https://aitoolguide.ai/blog/best-vector-databases-for-rag-2026/)

## Liên kết

- [02-embedding-models.md](02-embedding-models.md) — chọn model tạo vector để lưu vào DB ở trên.
- [04-rag-pipeline-tools.md](04-rag-pipeline-tools.md) — vị trí vector DB trong pipeline RAG tổng thể (mắt xích Retrieval).
- [03-rerankers.md](03-rerankers.md) — sau khi vector DB trả top-k, có thể rerank để tăng chất lượng.
