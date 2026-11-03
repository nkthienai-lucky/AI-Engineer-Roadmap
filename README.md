# AI Engineer Roadmap — LLM · VLM · RAG · GenAI

> Lộ trình học + xây sản phẩm, tối ưu cho người muốn **hiểu bản chất** và **deploy được sản phẩm thật với chi phí 0đ**.
> Tài liệu này là "một nguồn sự thật" — chốt hướng đi học và hướng đi xây sản phẩm cuối.

---

## 🎯 Mục tiêu sau lộ trình (18 tuần)

1. **Hiểu bản chất LLM/VLM** — tự code được transformer nhỏ, không phải "người dùng API".
2. **Xây được hệ thống RAG hoàn chỉnh** — kèm đánh giá (eval) có số liệu, không đoán mò.
3. **Xây sản phẩm GenAI hoàn chỉnh** — chat + RAG + hiểu ảnh, deploy lên hạ tầng free, có domain + DB + model chạy thật.
4. **Có portfolio** — 5 project chạy được + 1 capstone.

---

## 🗺️ Lộ trình tổng thể (18 tuần)

| Giai đoạn | Tuần | Chủ đề | Project đầu ra | Tài liệu |
|---|---|---|---|---|
| 1. Nền tảng | 1–3 | Python/API, GenAI overview, prompting | Chatbot CLI | [01-FOUNDATIONS](docs/cores/01-FOUNDATIONS/README.md) |
| 2. Bản chất LLM | 4–7 | Transformer, attention, tokenizer, fine-tuning | Mini-GPT train từ đầu | [02-LLM-INTERNALS](docs/cores/02-LLM-INTERNALS/README.md) |
| 3. RAG | 8–11 | Chunking, embedding, vector DB, rerank, **eval** | Docs-QA có trích dẫn | [03-RAG](docs/cores/03-RAG/README.md) |
| 4. VLM | 12–14 | CLIP, ViT, multimodal, VLM nhẹ | Hỏi đáp ảnh (SmolVLM) | [04-VLM](docs/cores/04-VLM/README.md) |
| 5. Agents & Production | 15–18 | Tool calling, agent, guardrails, observability | **Capstone: sản phẩm cuối** | [05-AGENTS-PRODUCTION](docs/cores/05-AGENTS-PRODUCTION/README.md) |

Song song suốt lộ trình: **deploy từng project lên web** (xem [07-DEPLOYMENT](docs/cores/07-DEPLOYMENT/README.md)).

---

## 📁 Cấu trúc thư mục

| File | Nội dung |
|---|---|
| [01-FOUNDATIONS](docs/cores/01-FOUNDATIONS/README.md) | Giai đoạn 1 — Nền tảng GenAI & prompting |
| [02-LLM-INTERNALS](docs/cores/02-LLM-INTERNALS/README.md) | Giai đoạn 2 — Transformer, attention, training |
| [03-RAG](docs/cores/03-RAG/README.md) | Giai đoạn 3 — RAG end-to-end + evaluation |
| [04-VLM](docs/cores/04-VLM/README.md) | Giai đoạn 4 — Vision-Language Models |
| [05-AGENTS-PRODUCTION](docs/cores/05-AGENTS-PRODUCTION/README.md) | Giai đoạn 5 — Agents, guardrails, production |
| [06-ECOSYSTEM](docs/cores/06-ECOSYSTEM/README.md) | Bảng tra nhanh: Vector DB, embedding, frameworks, free tiers |
| [07-DEPLOYMENT](docs/cores/07-DEPLOYMENT/README.md) | Kiến trúc & checklist deploy 0đ (Oracle Always Free) |
| [08-FINAL-PROJECT](docs/cores/08-FINAL-PROJECT/README.md) | **Spec sản phẩm cuối** — kiến trúc, milestones, tiêu chí nghiệm thu |

---

## 📏 Nguyên tắc học (đọc trước khi bắt đầu)

1. **Học theo project, không học lan man.** Mỗi giai đoạn PHẢI có project chạy được. Không hoàn thành project = không qua giai đoạn.
2. **Hiểu bản chất trước, dùng framework sau.** Xong GĐ2 phải tự code attention được bằng numpy/PyTorch; framework (LangChain...) chỉ là công cụ rút ngắn.
3. **Eval là kỹ năng đắt giá nhất.** Người build RAG giỏi khác người build qua loa ở chỗ: đo lường được chất lượng. Tập đánh giá từ GĐ3.
4. **Deploy ngay từ tuần 3.** Project không deploy = project chưa xong. Hạ tầng free đã có sẵn trong [07-DEPLOYMENT](docs/cores/07-DEPLOYMENT/README.md).
5. **Ghi chép công khai.** Mỗi project có README + demo link. Đây là portfolio khi đi xin việc/freelance.

## ✅ Định nghĩa "hoàn thành" lộ trình

- [ ] Tự code attention + transformer block chạy đúng (không nhìn tài liệu)
- [ ] RAG pipeline tự viết (không framework) + Ragas score > 0.8 faithfulness
- [ ] Deploy 1 sản phẩm GenAI lên domain riêng, có DB, model chạy trên server thật
- [ ] Trả lời được câu hỏi phỏng vấn: "Tại sao RAG của bạn retrieval kém? Bạn đo bằng gì?"

## ⏱️ Nhịp độ đề xuất

- **Toàn thời gian / sinh viên:** 4–5 giờ/ngày → xong trong 18 tuần (có thể rút còn 12 nếu bỏ bớt project GĐ2)
- **Đi làm:** 1–2 giờ/ngày + 4 giờ cuối tuần → 6–8 tháng
- **Muốn nhanh (3 tháng):** GĐ1 rút còn 1 tuần, GĐ2 học Karpathy chọn lọc (video 1–5), dồn thời gian cho GĐ3 + GĐ5
