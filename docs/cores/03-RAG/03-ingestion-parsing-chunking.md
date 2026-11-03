# Ingestion, Parsing & Chunking (Tuần 9 — nửa quan trọng nhất!)

> **Mục tiêu:** Biết parse tài liệu PDF/Word/HTML bằng `Docling` / `Unstructured` (vì sao PDF lộn xộn khó parse), nắm các chiến lược chunking (fixed-size, overlap, semantic, theo heading) với ví dụ chunk 512 overlap 64, hiểu vai trò của metadata (nguồn, trang, thời gian) trong filter + trích dẫn, và thuộc lòng indexing pipeline: document → chunk → embed → store.
> Nguồn mở rộng từ: [03-RAG.md](../../../03-RAG.md)

---

> 💡 Vì sao tuần này là "nửa quan trọng nhất"? Mọi cải tiến retrieval hay prompt đều **vô nghĩa nếu chunk rác**. Chunk quá to → pha loãng ngữ nghĩa, embedding không "bắt" được chủ đề cụ thể. Chunk quá nhỏ → mất ngữ cảnh, câu trả lời thiếu dữ kiện. Chất lượng ingestion quyết định trần chất lượng của toàn bộ RAG.

## Parse tài liệu: PDF, Word, HTML

### Vì sao PDF lộn xộn và khó parse

PDF được thiết kế để **hiển thị** (giữ vị trí từng glyph trên trang), không để lưu cấu trúc ngữ nghĩa. Vì vậy khi đọc ngược lại:

- **Bố cục 2 cột:** text bị đọc xen kẽ cột trái/phải thành một đoạn vô nghĩa.
- **Bảng:** ô bảng thành chuỗi text rời rạc; header mất liên kết với dữ liệu.
- **Header/footer/trang số:** lặp lại trên mọi trang, thành "noise" nếu không lọc.
- **Ảnh scan:** không có text layer — cần OCR.
- **Thứ tự đọc sai:** textbox, footnote, caption bị lẫn vào nội dung chính.

Word (`.docx`) dễ hơn vì có cấu trúc (heading, list, table XML), HTML có DOM tree — cả hai giữ ngữ nghĩa tốt hơn PDF nhiều.

### Hai thư viện nên dùng

- **`Docling`** (IBM): parse PDF/Word/HTML/ảnh thành tài liệu có cấu trúc (heading, bảng, list), giữ được layout, xuất Markdown — rất hợp pipeline RAG vì chunk theo heading trực tiếp được.
- **`Unstructured`**: "Swiss army knife" của ingestion — hỗ trợ rất nhiều định dạng, có chiến lược partition khác nhau cho từng loại file, tích hợp tốt với hệ sinh thái RAG.

```python
from docling.document_converter import DocumentConverter

doc = DocumentConverter().convert("handbook.pdf").document
md = doc.export_to_markdown()   # Markdown đã giữ heading/bảng → sẵn sàng chunk
```

Lời khuyên: đừng tự parse PDF bằng regex — hãy để thư viện chuyên dụng xử lý layout, rồi bạn tập trung vào chunking.

## Chiến lược chunking

Chunking = cắt tài liệu thành các đoạn (chunk) nhỏ để embed riêng từng đoạn. Bốn chiến lược chính:

### Fixed-size chunking

Cắt văn bản theo **số token/ký tự cố định**. Đơn giản nhất, hoạt động tốt khi văn bản đồng nhất (blog, sách). Nhược điểm: cắt giữa câu, giữa đoạn logic — mất ngữ cảnh.

### Overlap

Mỗi chunk **trùng lặp** một phần với chunk trước/sau. Khi một ý quan trọng nằm ở ranh giới hai chunk, overlap đảm bảo ý đó vẫn xuất hiện trọn vẹn trong ít nhất một chunk. Overlap thường bằng 10–20% chunk size.

**Ví dụ minh họa — chunk 512 token, overlap 64 token:**

```python
def chunk_fixed(text_tokens, chunk_size=512, overlap=64):
    step = chunk_size - overlap        # 448 token mỗi bước nhảy
    chunks = []
    for start in range(0, len(text_tokens), step):
        chunk = text_tokens[start:start + chunk_size]
        chunks.append(chunk)
        if start + chunk_size >= len(text_tokens):
            break
    return chunks
# Token 0–511 (chunk 1), token 448–959 (chunk 2): 64 token 448–511 nằm ở cả hai chunk
```

Chunk 1 chứa token 0–511, chunk 2 chứa token 448–959 — vùng 448–511 xuất hiện ở cả hai, nên một khái niệm nằm ở ranh giới vẫn nguyên vẹn trong ít nhất một chunk.

### Semantic chunking

Cắt theo **sự thay đổi ngữ nghĩa** thay vì độ dài: nhúng từng câu/đoạn nhỏ liên tiếp, tính độ tương đồng giữa hai đoạn kề nhau, và cắt tại nơi độ tương đồng **giảm mạnh** (chuyển chủ đề). Ưu điểm: mỗi chunk là một chủ đề liền mạch. Nhược: tốn embedding call hơn lúc ingestion, độ dài chunk không đều.

### Chunk theo heading

Tận dụng cấu trúc tài liệu: mỗi mục `##`/`###` (từ Markdown/Docling/HTML) thành một hoặc vài chunk. Giữ đúng ranh giới logic mà tác giả đã viết. Chiến lược này kết hợp tốt với fixed-size: chunk theo heading trước, nếu mục nào quá dài thì cắt nhỏ thêm bằng fixed-size.

**Khuyến nghị thực hành:** thử **256 / 512 / 1024** token và **đo** bằng eval (file 06) thay vì đoán. Văn bản pháp lý, FAQ hợp chunk nhỏ (256–512); tài liệu giải thích khái niệm hợp chunk to hơn (512–1024).

## Metadata: nguồn, trang, thời gian

Mỗi chunk phải mang metadata tối thiểu:

| Trường | Ví dụ | Dùng để |
|---|---|---|
| `source` / `doc_id` | `handbook.pdf` | Filter theo tài liệu; hiển thị nguồn |
| `page` | `trang 12` | Trích dẫn chính xác (trang X, đoạn Y) |
| `created_at` / `updated_at` | `2025-01-05` | Filter theo thời gian; ưu tiên tài liệu mới |
| `section` | `## Chính sách bảo mật` | Trích dẫn theo mục |

Hai công dụng bắt buộc:

1. **Filter khi truy vấn:** "chỉ tìm trong tài liệu A", "chỉ tài liệu cập nhật sau 2024" — kết hợp điều kiện metadata với vector search (xem [file 02](02-vector-db-pgvector-qdrant.md)).
2. **Trích dẫn nguồn:** câu trả lời phải trỏ được về `source + page` để người dùng click ra đoạn gốc — đây là DoD của project (file 07) và là nền tảng của citation (file 05).

## Indexing pipeline: document → chunk → embed → store

```
┌──────────┐   ┌──────────┐   ┌──────────────┐   ┌─────────────┐   ┌───────────┐
│ Document │──▶│  Parse   │──▶│    Chunk     │──▶│   Embed     │──▶│   Store   │
│ PDF/Word │   │ Docling/ │   │ fixed/seman./│   │ embedding   │   │ pgvector/ │
│ /HTML    │   │Unstruct. │   │ theo heading │   │ model       │   │  Qdrant   │
└──────────┘   └──────────┘   └──────────────┘   └─────────────┘   └───────────┘
                                   │                                    │
                                   └──── metadata (nguồn, trang, tgian) ─┘
```

1. **Document:** file thô (PDF, Word, HTML) được nạp vào.
2. **Parse:** chuyển thành text có cấu trúc (Markdown), lọc header/footer, xử lý bảng.
3. **Chunk:** cắt thành đoạn theo chiến lược ở trên, **gắn metadata** cho từng chunk.
4. **Embed:** gọi embedding model, mỗi chunk → 1 vector.
5. **Store:** ghi cả `content + embedding + metadata` vào Vector DB (upsert theo `doc_id` để cập nhật tài liệu cũ).

Pipeline này cần **chạy được lặp lại** (re-index khi tài liệu đổi) và **idempotent** (chạy lại không nhân đôi dữ liệu) — hãy thiết kế `doc_id + chunk_id` ổn định ngay từ đầu.

## Liên kết

- [01 — Embedding & Vector Search](01-embedding-and-vector-search.md) — bước "embed" dùng model gì
- [02 — Vector DB: pgvector & Qdrant](02-vector-db-pgvector-qdrant.md) — bước "store" vào đâu
- [04 — Retrieval & Hybrid Search](04-retrieval-reranking-hybrid.md) — chunk tốt thì retrieval mới tốt
- [06 — Evaluation với Ragas](06-evaluation-ragas.md) — đo chunk size nào tốt nhất
