# 02 — Embedding Models: bảng tra & cách chọn

> **Mục tiêu**: chọn model embedding cho RAG — hiểu cột "chất lượng"/"chạy CPU" đánh giá theo tiêu chí gì, và dùng MTEB Leaderboard để ra quyết định thay vì học thuộc bảng.
>
> Nguồn mở rộng từ: [06-ECOSYSTEM.md](../../../06-ECOSYSTEM.md)

---

## Bảng tra nhanh

| Model | Open source | Chất lượng | Chạy CPU? | Ghi chú |
|---|---|---|---|---|
| **Qwen3-Embedding-0.6B** | ✅ Apache 2.0 | Rất tốt | ✅ Mượt | **Best pick cho CPU / tiếng Việt** |
| **BGE-M3** (BAAI) | ✅ | Tốt, đa ngôn ngữ | ✅ OK | Đa ngôn ngữ mạnh |
| **nomic-embed-text-v1.5** | ✅ | Tốt | ✅ Rất nhẹ | Hay dùng với Ollama |
| Qwen3-Embedding-8B | ✅ | Top đầu | ⚠️ Cần GPU | Khi có GPU |
| OpenAI text-embedding-3 | ❌ | Tốt | — | $0.13/1M tokens |

---

## Cột "Chất lượng" đánh giá theo gì?

- "Chất lượng" ở đây nói gọn về **điểm retrieval trên benchmark MTEB** (đa phần là task Retrieval) + cảm nhận thực tế với dữ liệu tiếng Việt. Model "Rất tốt" (Qwen3-Embedding-0.6B) đạt điểm cao bất thường so với kích thước nhỏ — nên thành best pick.
- **Open source ✅** nghĩa là tải trọng số về chạy local, không lo giá/riêng tư dữ liệu; bản Apache 2.0 (Qwen3-Embedding-0.6B) còn thoải mái dùng thương mại. OpenAI text-embedding-3 là API đóng — chất lượng tốt nhưng phải trả $0.13/1M tokens và gửi dữ liệu ra ngoài.
- Trade-off cốt lõi: **chất lượng vs kích thước**. 0.6B gần bằng 8B trên nhiều benchmark nhưng chạy CPU mượt; 8B chỉ hơn đáng kể khi có GPU và nhu cầu cực khắt khe (top đầu bảng xếp hạng).

## Cột "Chạy CPU?" nghĩa là gì?

- Model embedding phải chạy **mọi lúc** (mỗi khi ingest tài liệu và mỗi khi query) — nếu bắt buộc GPU thì chi phí vận hành tăng mạnh. Cột này trả lời: có chạy được trên CPU (server nhỏ/VPS) không?
- ✅ Mượt / Rất nhẹ = tải trọng số vài trăm MB–1GB, encode đủ nhanh cho CPU. "⚠️ Cần GPU" = model 8B quá nặng cho CPU ở throughput thực tế. "—" với OpenAI vì chạy trên server của họ, không liên quan CPU/GPU của bạn.
- Mẹo: model nhỏ nhưng dữ liệu của bạn là tiếng Việt → hãy tự đo trên tập query thật, đừng chỉ tin benchmark tiếng Anh.

---

## MTEB Leaderboard là gì? Dùng để chọn model thế nào?

- **MTEB** (Massive Text Embedding Benchmark) là bảng xếp hạng chuẩn cho embedding models — chấm điểm nhiều task: Retrieval, Clustering, Classification, Semantic Textual Similarity... trên nhiều ngôn ngữ. Link trong nguồn: [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard).
- Quy trình chọn trên leaderboard:
  1. Lọc theo **task Retrieval** (nếu dùng cho RAG) — đừng nhìn điểm Overall vì pha lẫn task bạn không dùng.
  2. Lọc **multilingual / tiếng Việt** nếu tài liệu là tiếng Việt — model top bảng nhưng thiếu tiếng Việt sẽ hỏng dấu, hỏng từ.
  3. So **kích thước model / memory** với phần cứng của bạn (CPU → 0.6B trở xuống; GPU → 8B).
  4. Chọn 2–3 ứng viên, **test trên chính dữ liệu của mình** (10–20 query thật, so kết quả top-5) rồi mới chốt.
- Thứ hạng leaderboard đổi theo thời gian model mới ra — hãy tra live lúc quyết định, đừng học thuộc thứ hạng.

---

## Ghi chú từng dòng (ngắn)

- **Qwen3-Embedding-0.6B**: best pick cho CPU/tiếng Việt — Apache 2.0, chất lượng "rất tốt" dù chỉ 0.6B.
- **BGE-M3**: mạnh về đa ngôn ngữ, chạy CPU OK — hợp corpus nhiều thứ tiếng.
- **nomic-embed-text-v1.5**: rất nhẹ, hay đi kèm Ollama — hợp máy yếu/prototype nhanh.
- **Qwen3-Embedding-8B**: dùng khi có GPU và cần chất lượng top đầu.
- **OpenAI text-embedding-3**: khi không muốn tự host, chấp nhận $0.13/1M tokens.

## Liên kết

- [01-vector-db.md](01-vector-db.md) — vector sinh ra bởi các model này lưu vào đâu.
- [05-model-quick-picks.md](05-model-quick-picks.md) — bảng model chạy CPU/server nhỏ (gồm cả Qwen3-Embedding-0.6B).
- [04-rag-pipeline-tools.md](04-rag-pipeline-tools.md) — embedding nằm ở mắt xích nào của pipeline RAG.
