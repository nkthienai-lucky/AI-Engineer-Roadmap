# Kiến trúc & luồng xử lý — GenAI Knowledge Assistant

> **Mục tiêu:** Đọc và hiểu được sơ đồ kiến trúc tổng thể (User → Cloudflare → Oracle VPS), hiểu tường tận từng bước của 2 luồng lõi — RAG (PDF → trả lời có citation) và VLM (ảnh → JSON) — bao gồm tham số cụ thể (chunk 512, overlap 64, top-20 → top-5) và lý do đằng sau mỗi lựa chọn.
> Nguồn mở rộng từ: [08-FINAL-PROJECT](README.md)

---

## Sơ đồ kiến trúc

```
                        ┌──────────────────────────────────────┐
User ──HTTPS──► Cloudflare ──►│  Oracle Always Free VPS (2CPU/12GB) │
                        │                                      │
                        │  nginx ──► FastAPI backend            │
                        │             │                         │
                        │   ┌─────────┼──────────────┐          │
                        │   ▼         ▼              ▼          │
                        │ Ollama   pgvector     SmolVLM2        │
                        │ (Qwen    (Postgres)   (ảnh → text)    │
                        │ 2.5-3B)      │                         │
                        │   ▲     ┌────┴─────┐                  │
                        │   │     ▼          ▼                  │
                        │   │  bge-embed  bge-reranker          │
                        │   │  0.6B       (CPU)                 │
                        └───┼──────────────────────────────────┘
                            │
                     Langfuse (trace, optional)
```

### Đọc sơ đồ từ trên xuống

- **User → Cloudflare:** mọi request đi qua Cloudflare trước (proxy + DNS + TLS). Cloudflare lo HTTPS free, ẩn IP VPS khỏi internet, và là nơi hứng bớt traffic bất thường. Đổi lại, Cloudflare có timeout cho request dài — chính là nguồn gốc rủi ro 524 khi model sinh chậm (xem [06-acceptance-and-risks.md](06-acceptance-and-risks.md)).
- **nginx → FastAPI:** nginx làm reverse proxy, serve static (frontend build) và chuyển request API vào FastAPI. FastAPI là não điều phối: nhận request, gọi Ollama/pgvector/SmolVLM2, stream kết quả trả về qua SSE.
- **Ollama (Qwen2.5-3B):** LLM sinh câu trả lời bằng tiếng Việt, chạy CPU. Ollama lo nạp model, quantize, cấp phát RAM — backend chỉ gọi API.
- **pgvector (PostgreSQL):** vừa là DB quan hệ (users, conversations, feedback) vừa là vector store (chunk embedding). Một service duy nhất = ít thứ phải bảo trì hơn trên VPS 12GB.
- **SmolVLM2 (ảnh → text):** VLM riêng, xử lý luồng ảnh. Tách khỏi LLM để mỗi model tối ưu cho việc của mình.
- **bge-embed 0.6B + bge-reranker:** embedding model (văn bản → vector) và reranker (chấm lại độ liên quan) — hai mắt xích của retrieval, xem chi tiết bên dưới.
- **Langfuse (optional):** trace từng request (prompt, context, latency, token) — "optional" vì hệ thống vẫn chạy được nếu không có, nhưng thiếu nó thì story 5 (log chi phí) phải tự build.

Toàn bộ nằm trong **Oracle Always Free VPS (2CPU/12GB)** — chi phí 0đ, chi tiết deploy ở [07-DEPLOYMENT](../07-DEPLOYMENT/README.md).

## Luồng RAG

```
PDF → Docling parse → chunk 512 (overlap 64) → embed (Qwen3-Embed-0.6B) → pgvector
→ [query embed → top-20 → rerank top-5] → prompt + context → Qwen2.5-3B → trả lời + citation
```

### Phần ingest (offline, chạy khi upload)

1. **PDF → Docling parse:** Docling biến PDF lộn xộn (bảng, cột, header) thành markdown sạch. Bước này quyết định chất lượng của **toàn bộ** pipeline sau: parse sai thì chunk sai, retrieval sai, trả lời sai — "rác vào rác ra" theo nghĩa đen. Docling được chọn vì giữ được cấu trúc bảng/heading mà parser text thuần làm mất (xem [04-tech-stack.md](04-tech-stack.md)).
2. **Chunk 512, overlap 64:** mỗi đoạn 512 token, hai đoạn liền kề chồng nhau 64 token. 512 là điểm cân bằng: đủ lớn để giữ trọn một ý (một điều khoản, một đoạn giải thích), đủ nhỏ để embedding "đặc" (nhiều ý trong 1 vector sẽ làm loãng độ tương đồng). Overlap 64 tránh trường hợp một ý bị cắt đôi ở ranh giới chunk — phần "đầu" của ý nằm chunk trước, phần "cuối" vẫn còn trong overlap của chunk sau.
3. **Embed (Qwen3-Embed-0.6B):** mỗi chunk → 1 vector. Model 0.6B đa ngôn ngữ, chạy CPU mượt — đủ nhanh để embed hàng nghìn chunk mà không chiếm hết RAM của LLM.
4. **→ pgvector:** vector + metadata (document_id, page, chunk_id, text) lưu vào bảng có cột vector. Metadata chính là nơi citation "sống" — không lưu page/chunk_id ở bước này thì bước 7 không thể trích dẫn.

### Phần query (online, chạy khi hỏi)

5. **Query embed:** câu hỏi cũng được embed **bằng đúng model đó** — vector chỉ so sánh được khi cùng model, cùng chiều.
6. **Top-20:** truy vấn pgvector lấy 20 chunk gần nhất theo cosine similarity. Lấy 20 thay vì 5 ở bước đầu để "lưới rộng" — recall cao, chấp nhận lẫn lộn vài chunk không liên quan, vì bước sau sẽ lọc.
7. **Rerank top-5:** bge-reranker chấm lại độ liên quan **cặp (câu hỏi, chunk)** chính xác hơn embedding similarity (cross-encoder đọc cả hai cùng lúc thay vì nén mỗi thứ thành 1 vector). Giữ 5 chunk tốt nhất — đủ context cho câu trả lời mà không vượt context window, không làm loãng prompt.
8. **Prompt + context:** ghép 5 chunk thành context, kèm rule: "chỉ trả lời dựa trên context, trích dẫn [số thứ tự chunk]; nếu không có thông tin thì nói không tìm thấy". Đánh số `[1]`–`[5]` để model trích dẫn dễ và chính xác.
9. **Qwen2.5-3B → trả lời + citation:** model sinh câu trả lời streaming, mỗi câu trích dẫn gắn `[n]` → backend map `n` về `(document_id, page, chunk_id)` đã lưu ở bước 4 → trả citation kèm câu trả lời.

## Luồng ảnh

```
ảnh → SmolVLM2-2.2B → caption/JSON → (tuỳ chọn) đưa vào RAG context
```

1. **Ảnh → SmolVLM2-2.2B:** user upload ảnh hóa đơn/biểu mẫu; backend gửi ảnh + prompt yêu cầu trích xuất theo schema (ví dụ `{vendor, date, total}`).
2. **Caption/JSON:** model trả về hoặc caption mô tả (ảnh thường) hoặc JSON có cấu trúc (hóa đơn) — parse và validate JSON, trường thiếu thì để `null`, không đoán bừa.
3. **(Tuỳ chọn) đưa vào RAG context:** caption/JSON có thể được embed và đưa vào kho vector như 1 "chunk ảnh" — khi đó hỏi "tuần trước tôi mua gì?" sẽ tìm được cả thông tin từ hóa đơn. Đây là tính năng mở rộng, **không bắt buộc** ở M3 — tránh phình scope.

Hai luồng dùng chung FastAPI nhưng **tài nguyên tách**: VLM load theo demand, unload khi không dùng, để tránh OOM (xem rủi ro OOM ở [06-acceptance-and-risks.md](06-acceptance-and-risks.md)).

## Liên kết

- [04 — Tech Stack](04-tech-stack.md) — lý do chọn từng thành phần trong sơ đồ
- [05 — Milestones](05-milestones.md) — luồng nào lên ở milestone nào (RAG = M2, ảnh = M3)
- [06 — Acceptance & rủi ro](06-acceptance-and-risks.md) — đo lường và rủi ro của từng luồng
