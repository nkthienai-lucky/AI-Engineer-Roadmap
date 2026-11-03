# AI Engineer Roadmap — Docs (bản chi tiết)

> Bản **mở rộng chi tiết** từ các file gốc trong [`AI-Engineer-Roadmap/`](../README.md).
> Mỗi giai đoạn (01–08) = 1 folder trong [`cores/`](cores/), mỗi folder gồm nhiều file theo chủ đề để học từ đầu.

## 📖 Đọc theo thứ tự nào?
1. **[00-TONG-HOP.md](00-TONG-HOP.md)** — bản đồ toàn cảnh (đọc trước 1 lần, quay lại khi lạc)
2. Học lần lượt `cores/01` → `cores/05`; `cores/06` là bảng **tra**, `cores/07` dùng khi deploy, `cores/08` là spec capstone
3. Mỗi folder: đọc theo số thứ tự file (01 → 0N), file `0N-resources-and-common-pitfalls.md` đọc cuối mỗi giai đoạn

## 📁 Cấu trúc

| Folder | Giai đoạn | File |
|---|---|---|
| [cores/01-FOUNDATIONS](cores/01-FOUNDATIONS/) | Nền tảng GenAI & Prompting (tuần 1–3) | 6 file: Python/Git/REST · khái niệm GenAI · prompting · Ollama & API · project chatbot · tài nguyên |
| [cores/02-LLM-INTERNALS](cores/02-LLM-INTERNALS/) | Bản chất LLM (tuần 4–7) | 8 file: BPE/embedding · positional encoding/RoPE · self-attention · kiến trúc transformer · training · LoRA/quantization · project Mini-GPT · tài nguyên |
| [cores/03-RAG](cores/03-RAG/) | RAG end-to-end (tuần 8–11) | 8 file: embedding/vector search · vector DB · ingestion/chunking · retrieval/rerank/hybrid · generation/citation · eval Ragas · project Docs-QA · tài nguyên |
| [cores/04-VLM](cores/04-VLM/) | VLM & Multimodal (tuần 12–14) | 7 file: CNN/ViT · CLIP/visual tokens · kiến trúc VLM · thực hành/prompt · multimodal RAG/OCR · project trợ lý ảnh · tài nguyên |
| [cores/05-AGENTS-PRODUCTION](cores/05-AGENTS-PRODUCTION/) | Agents & Production (tuần 15–18) | 7 file: tool calling · agent loop/ReAct · memory/MCP · guardrails · observability · production hóa/capstone · tài nguyên |
| [cores/06-ECOSYSTEM](cores/06-ECOSYSTEM/) | Bảng tra công cụ | 6 file: vector DB · embedding · rerankers · pipeline tools · model CPU · hạ tầng tính tiền |
| [cores/07-DEPLOYMENT](cores/07-DEPLOYMENT/) | Deploy 0đ Oracle Always Free | 7 file: kiến trúc · Oracle · domain/SSL/monitoring · bẫy billing & firewall · streaming & RAM · bảo mật/bền vững · setup & pre-flight |
| [cores/08-FINAL-PROJECT](cores/08-FINAL-PROJECT/) | Spec capstone | 7 file: tầm nhìn/scope · user stories · kiến trúc & luồng · tech stack · milestones · acceptance & rủi ro · sau khi hoàn thành |

## 🔗 Quy ước link
- Mỗi file có dòng `> Nguồn mở rộng từ: ...` trỏ về **file gốc** trong `AI-Engineer-Roadmap/` — đối chiếu khi cần bản đầy đủ.
- Link `../../../0X-....md` = file gốc; link `../0X-FOLDER/...` = file khác trong `cores/`.
