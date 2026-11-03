# Vector DB: pgvector & Qdrant (Tuần 8)

> **Mục tiêu:** Hiểu Vector DB là gì và tại sao cần nó thay vì quét toàn bộ dữ liệu, thành thạo **1 trong 2** — `pgvector` (khuyên dùng) hoặc `Qdrant` — kèm schema và truy vấn cơ bản, và biết vị trí của chúng trong bức tranh tổng quan hệ sinh thái.
> Nguồn mở rộng từ: [03-RAG.md](../../../03-RAG.md)

---

## Vector DB là gì, tại sao cần

Vector DB là cơ sở dữ liệu được thiết kế để **lưu vector embedding và truy vấn "k vector gần nhất"** (top-k nearest neighbors) một cách hiệu quả. Về bản chất, nó làm hai việc:

1. **Lưu trữ:** giữ hàng triệu đến hàng tỷ vector cùng metadata (nguồn, trang, thời gian…).
2. **Truy vấn ngữ nghĩa:** với vector câu hỏi, trả về các chunk có nghĩa gần nhất — kèm điều kiện filter metadata.

### Tại sao không quét toàn bộ (brute-force)?

Với 1 triệu vector 768 chiều, mỗi truy vấn brute-force phải tính 1 triệu phép so sánh — vài chục đến vài trăm ms trên CPU, và tốn RAM băng thông khổng lồ khi dữ liệu lớn dần. Ở quy mô production (10–100 triệu vector), cách này không khả thi về cả latency lẫn chi phí.

Vector DB giải quyết việc này bằng **ANN index** (HNSW, IVF — xem [file 01](01-embedding-and-vector-search.md)): dựng cấu trúc tìm kiếm cho phép bỏ qua phần lớn dữ liệu mà vẫn tìm được "gần đúng đủ tốt". Ngoài ra, Vector DB còn cung cấp:

- **Metadata filtering:** "chỉ tìm trong tài liệu X", "chỉ chunk từ tuần trước" — kết hợp filter với similarity search.
- **Quản lý index:** build, cập nhật (upsert), xóa vector mà không phải tự lo cấu trúc dữ liệu.
- **Độ bền & mở rộng:** lưu đĩa, nhân bản (replica), phân mảnh (shard) khi dữ liệu lớn.

Nói ngắn gọn: nếu bạn tự lưu vector trong list Python và quét tay, mọi thứ sẽ vỡ khi dữ liệu vượt vài chục nghìn chunk. Vector DB sinh ra để tránh điều đó.

## Lựa chọn 1: pgvector (khuyên dùng)

`pgvector` là **extension của PostgreSQL** — nghĩa là bạn dùng đúng database quen thuộc, thêm biến thể vector vào như một kiểu dữ liệu mới (`vector(768)`), và query bằng SQL thuần. Đây là lựa chọn **khuyên dùng** cho phần lớn dự án, đặc biệt khi:

- Bạn **đã có PostgreSQL** trong stack — không thêm hạ tầng mới, không thêm service phải vận hành.
- Dữ liệu phi cấu trúc (vector) cần **truy vấn chung với dữ liệu có cấu trúc** (users, đơn hàng, quyền truy cập) trong cùng transaction/JOIN.
- Quy mô vừa (dưới vài triệu vector) — HNSW của pgvector đủ nhanh cho hầu hết use-case.
- Muốn backup, replication, kiểm soát quyền… đều dùng tooling Postgres sẵn có.

### Ví dụ schema và truy vấn cơ bản

```sql
-- Bật extension (một lần)
CREATE EXTENSION IF NOT EXISTS vector;

-- Bảng lưu chunk đã embed (768 chiều)
CREATE TABLE document_chunks (
    id          bigserial PRIMARY KEY,
    doc_id      text NOT NULL,        -- nguồn: file/tài liệu nào
    page        int,                  -- trang (để trích dẫn)
    content     text NOT NULL,        -- nội dung chunk
    embedding   vector(768) NOT NULL,
    created_at  timestamptz DEFAULT now()
);

-- Index ANN (HNSW, metric cosine)
CREATE INDEX ON document_chunks
    USING hnsw (embedding vector_cosine_ops);

-- Truy vấn: 5 chunk gần nhất, chỉ trong 1 tài liệu
SELECT id, doc_id, page, content,
       1 - (embedding <=> :query_vec) AS similarity
FROM document_chunks
WHERE doc_id = :doc_id
ORDER BY embedding <=> :query_vec
LIMIT 5;
```

Trong đó: `<=>` là toán tử cosine distance, `<->` là L2, `<#>` là negative inner product — phải dùng đúng operator khớp với index (`vector_cosine_ops` ↔ `<=>`).

## Lựa chọn 2: Qdrant

Qdrant là **vector database chuyên dụng** viết bằng Rust, chạy như service riêng (Docker hoặc Qdrant Cloud). Chọn Qdrant khi:

- **Performance cao** là ưu tiên số 1: độ trễ thấp ổn định ở quy mô lớn (hàng chục triệu vector trở lên), throughput cao.
- Cần **filter mạnh**: Qdrant cho phép filter metadata rất linh hoạt (filter kết hợp điều kiện trên payload, kể cả điều kiện trên chính vector index) mà không tách thành 2 bước query-then-filter.
- Muốn tách tầng vector search khỏi database chính, hoặc không muốn "gánh" index vector trong Postgres production.

Ví dụ truy vấn cơ bản với client Python:

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct, Filter, FieldCondition, MatchValue

client = QdrantClient(url="http://localhost:6333")

client.create_collection(
    collection_name="docs",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
)

client.upsert("docs", points=[
    PointStruct(id=1, vector=[0.1] * 768,
                payload={"doc_id": "handbook.pdf", "page": 3, "content": "..."})
])

hits = client.query_points(
    "docs",
    query=[0.1] * 768,
    query_filter=Filter(must=[FieldCondition(key="doc_id", match=MatchValue(value="handbook.pdf"))]),
    limit=5,
).points
```

## pgvector vs Qdrant vs giải pháp khác

| Tiêu chí | pgvector | Qdrant |
|---|---|---|
| Hình thái | Extension Postgres | Service riêng (Rust) |
| Khi nào dùng | Đã có Postgres; dữ liệu vector + quan hệ chung hệ thống | Performance cao, filter mạnh, quy mô lớn |
| Query | SQL thuần | Client API (Python, REST…) |
| Vận hành | Không thêm service mới | Thêm service (Docker/cloud) |
| Filter metadata | WHERE SQL (mạnh, JOIN được) | Payload filter (linh hoạt, native) |
| Khuyến nghị trong roadmap | ✅ **Khuyên dùng** | ✅ Khi cần performance chuyên dụng |

Để tra nhanh bảng so sánh đầy đủ Vector DB / embedding / free tier, xem [06-ECOSYSTEM.md](../../../06-ECOSYSTEM.md) và mục Vector DB chi tiết tại [06-ECOSYSTEM/01-vector-db.md](../../cores/06-ECOSYSTEM/01-vector-db.md) — đó là tài liệu tham chiếu hệ sinh thái chính thức của bộ roadmap này.

## Liên kết

- [01 — Embedding & Vector Search](01-embedding-and-vector-search.md) — cosine/HNSW/IVF mà Vector DB dùng ở dưới lớp
- [03 — Ingestion & Chunking](03-ingestion-parsing-chunking.md) — pipeline nạp dữ liệu vào bảng/collection trên
- [07 — Project Docs-QA](07-project-docs-qa.md) — V1 của project dùng pgvector hoặc Qdrant
