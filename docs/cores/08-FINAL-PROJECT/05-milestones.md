# Milestones — GenAI Knowledge Assistant

> **Mục tiêu:** Nắm 4 milestone M1–M4 (tuần 15–18), nội dung và tiêu chí nghiệm thu từng milestone, quy tắc cứng "M1–M3 local + HF Spaces, M4 mới deploy production", và cách capstone ghép với 5 giai đoạn học trước đó (GĐ1–5).
> Nguồn mở rộng từ: [08-FINAL-PROJECT](README.md)

---

## Bảng milestones (giữ nguyên từ spec)

| Milestone | Tương ứng | Nội dung | Nghiệm thu |
|---|---|---|---|
| **M1** (tuần 15) | GĐ1–2 xong | Chat core: FastAPI + Ollama, streaming, history | Chat hoạt động trên web |
| **M2** (tuần 16) | GĐ3 xong | RAG: ingest PDF → hỏi đáp có citation | Trích dẫn đúng trang; Ragas faithfulness > 0.8 |
| **M3** (tuần 17) | GĐ4 xong | VLM: upload ảnh → mô tả / trích JSON | Trích xuất đúng 8/10 hóa đơn test |
| **M4** (tuần 18) | GĐ5 xong | Production: deploy Oracle, monitoring, agent tool (tìm kiếm web/tính toán) | Site live trên domain, uptime > 99% tuần đầu |

## Chi tiết từng milestone

### M1 — Chat core (tuần 15, sau GĐ1–2)
Đây là "xương sống" mọi milestone sau: FastAPI backend + Ollama (Qwen2.5-3B) + streaming SSE + lưu lịch sử hội thoại + UI chat tối thiểu. Khi M1 xong, bạn có US3 (chat giữ ngữ cảnh) chạy hoàn chỉnh. Nghiệm thu "chat hoạt động trên web" nghĩa là: mở trình duyệt, chat được, thấy chữ stream ra từng token, reload trang vẫn còn lịch sử. **Không** có RAG, không có ảnh — chỉ chat.

### M2 — RAG + citation (tuần 16, sau GĐ3)
Nguyên luồng RAG: ingest PDF (Docling → chunk 512/overlap 64 → embed → pgvector), retrieval (top-20 → rerank top-5), generation có citation. Đây là milestone **khó nhất và quan trọng nhất** — vì "cốt lõi phải giỏi" là RAG + Eval. Nghiệm thu gồm 2 vế, cả 2 đều bằng số liệu: **trích dẫn đúng trang** (test tay trên câu hỏi biết sẵn đáp án) và **Ragas faithfulness > 0.8** (chạy bộ 30 câu test — xem [06-acceptance-and-risks.md](06-acceptance-and-risks.md)). Đạt M2 là US1 xong.

### M3 — VLM trích JSON (tuần 17, sau GĐ4)
Thêm SmolVLM2: upload ảnh hóa đơn/biểu mẫu → JSON có cấu trúc. Nghiệm thu "trích xuất đúng 8/10 hóa đơn test" — bộ 10 ảnh này **tự chuẩn bị từ đầu M3**, đa dạng độ rõ/bố cục, để không "teach to the test" với ảnh dễ. Đạt M3 là US2 xong. Lưu ý: phần "đưa caption vào RAG context" là tuỳ chọn, không thuộc nghiệm thu M3 (xem [03-architecture-and-flows.md](03-architecture-and-flows.md)).

### M4 — Production + agent tool (tuần 18, sau GĐ5)
Ba việc: (1) deploy Oracle VPS + Cloudflare + domain, (2) monitoring (Langfuse + UptimeRobot, US5), (3) agent tool tối thiểu (tìm kiếm web / tính toán). Nghiệm thu: site live trên domain (is-a.dev hoặc domain riêng), uptime > 99% tuần đầu. Agent tool ở mức "gọi được 1–2 tool", không xây framework agent — M4 là nơi **chứng minh chạy production**, không phải nơi mở rộng tính năng.

## Quy tắc cứng

> **M1–M3 chạy local + HF Spaces; M4 mới deploy production. Không mở rộng scope ngoài 4 milestone.**

- **M1–M3 local + HF Spaces:** local để iterate nhanh (sửa 1 dòng → test ngay, không cần build image/push/wait deploy). HF Spaces dùng khi cần demo cho người khác xem trước M4 — deploy "demo-grade" chứ không phải production. Tách bạch hai thứ này để không mất thời gian lo TLS/nginx/uptime khi cái cần tối ưu là prompt và retrieval.
- **M4 mới deploy production:** đúng 1 lần lo hạ tầng, khi code đã ổn. Xem [07-DEPLOYMENT](../07-DEPLOYMENT/README.md).
- **Không mở rộng scope:** mọi ý tưởng tính năng mới trong M1–M4 → ghi backlog. Mục tiêu là **hoàn thành** 4 milestone, không phải "làm được nhiều nhất có thể". Chi tiết cách chống cám dỗ: rủi ro "quá ham tính năng mới" ở [06-acceptance-and-risks.md](06-acceptance-and-risks.md).

## Ghép với lộ trình học GĐ1–5

Lộ trình 18 tuần chia 5 giai đoạn (xem [README.md](../../../README.md)); capstone chiếm **4 tuần cuối**, và mỗi milestone là "bài thi" của giai đoạn học tương ứng:

| Giai đoạn học | Kiến thức cốt lõi | Hiểu tại M nào trong capstone |
|---|---|---|
| GĐ1 — Nền tảng GenAI & prompting | prompt, token, context, Ollama | M1: prompt cho chat, quản lý context window |
| GĐ2 — Transformer, attention, training | hiểu bản chất model chạy thế nào | M1: tối ưu latency CPU, quantize Q4, max_tokens |
| GĐ3 — RAG + evaluation | ingestion, retrieval, rerank, Ragas | M2: toàn bộ luồng RAG + eval faithfulness |
| GĐ4 — VLM | hiểu ảnh, caption, trích xuất | M3: SmolVLM2 → JSON |
| GĐ5 — Agents, guardrails, production | agent tool, guardrail, deploy, monitoring | M4: deploy + agent tool + guardrail citation |

Cách học đúng: GĐ1–2 xây nền (mỗi giai đoạn có project riêng), GĐ3–5 thì project giai đoạn **gần như là một phần của capstone** — làm project GĐ3 xong thì khối RAG của M2 gần xong. Nhờ vậy 4 tuần capstone chủ yếu là ghép mảnh + hardening, không phải viết từ đầu.

## Quy trình "nhận" một milestone

Mỗi milestone chỉ tính là xong khi cả 3 vế sau đúng — thói quen này chính là discipline của production:

1. **Chạy được end-to-end trên kịch bản thật** (không phải unit test từng phần ghép vội): ví dụ M2 là upload 1 PDF thật → hỏi → thấy citation.
2. **Đạt tiêu chí nghiệm thu định lượng** trong bảng trên (faithfulness > 0.8, 8/10 hóa đơn…), chạy bằng script để lặp lại được.
3. **Không phá milestone trước** — regression check nhanh: chat M1 vẫn streaming tốt sau khi thêm RAG ở M2.

Nếu trượt milestone, xử lý theo thứ tự: giảm max_tokens / đơn giản hoá prompt trước, tối ưu retrieval sau, và **cắt tính năng phụ** — chứ không kéo dài tuần. Kéo dài 1 milestone là dồn toàn bộ M4 (production) vào cuối, nơi không còn đường lùi.

## Liên kết

- [02 — User Stories](02-user-stories.md) — story nào nghiệm thu ở milestone nào (US3→M1, US1→M2, US2→M3, US4/US5→M4)
- [06 — Acceptance & rủi ro](06-acceptance-and-risks.md) — nghiệm thu tổng cuối cùng sau M4
- [04 — Tech Stack](04-tech-stack.md) — công cụ cần sẵn cho từng milestone
