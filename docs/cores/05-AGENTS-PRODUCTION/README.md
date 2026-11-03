# Giai đoạn 5 — Agents & Production (Tuần 15–18)

> **Mục tiêu:** Ghép mọi thứ thành sản phẩm hoàn chỉnh: agent có tool, an toàn, quan sát được, deploy production.
> **Đầu ra:** **Sản phẩm cuối (capstone)** — xem spec chi tiết tại [08-FINAL-PROJECT](../08-FINAL-PROJECT/README.md).

---

## 🎓 Nội dung cần học

### Tuần 15 — Tool calling & Agents
- Function/tool calling: cơ chế, JSON schema, error handling
- Agent loop: ReAct (reason + act), planner, executor
- Memory: ngắn hạn (context) vs dài hạn (vector store)
- Framework: **LangGraph** (khuyên dùng) hoặc function-calling thuần
- MCP (Model Context Protocol) — đọc khái niệm (chuẩn kết nối tool mới)

### Tuần 16 — An toàn & Guardrails
- Prompt injection: nhận diện + phòng chống (qua RAG, qua user input)
- Output guardrails: schema validation, content filter
- Tách bạch "system prompt / dữ liệu / input user" trong RAG
- Rate limit, auth, chi phí per-request

### Tuần 17 — Observability & Eval production
- Logging trace: **Langfuse** (self-host free trên Oracle VPS)
- Eval online: theo dõi chất lượng theo thời gian thật
- Theo dõi chi phí token, latency P95
- A/B thử prompt/model khác nhau

### Tuần 18 — Production hóa + Capstone
- Docker hóa toàn bộ stack, deploy lên Oracle Always Free
- CI/CD cơ bản (GitHub Actions)
- Load test nhẹ, tối ưu (KV cache, batching, caching semantic)
- Viết README + demo video cho portfolio

---

## 📚 Tài nguyên

| Tài nguyên | Nội dung | Link |
|---|---|---|
| **DeepLearning.AI — Context Engineering** | Cho agent đúng thông tin, đúng lúc | [deeplearning.ai/courses/context-engineering](https://www.deeplearning.ai/courses/context-engineering) |
| **LangGraph docs** | Xây agent có trạng thái | [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/) |
| **Full Stack Deep Learning** | Deploy LLM production | [fullstackdeeplearning.com](https://fullstackdeeplearning.com/) |
| **AI Engineer Roadmap 2026** | Toàn cảnh kỹ năng + ý tưởng project | [dataskew.io/roadmaps/ai-engineering](https://dataskew.io/roadmaps/ai-engineering/) |
| **Langfuse** | Observability (self-host) | [langfuse.com](https://langfuse.com/) |

---

## 🛠️ Project GĐ5 = Capstone

Chính là **sản phẩm cuối** trong [08-FINAL-PROJECT](../08-FINAL-PROJECT/README.md) — tổng hợp LLM + RAG + VLM + agent, deploy lên hạ tầng free hoàn toàn.

---

## ⚠️ Lỗi thường gặp
- Agent "ảo tưởng quyền lực": cho agent quá nhiều tool không cần thiết → chậm, tốn token, hay sai. Chỉ cho tool thật sự cần
- Không có eval khi đổi prompt → cải tiến ảo
- Bỏ qua chi phí token khi design → sản phẩm chết vì hóa đơn. Tính chi phí/người dùng/ngày từ đầu
- Deploy "chạy trên máy mình" và gọi là xong → phải deploy lên server thật, có domain, có monitoring

---

## 🔗 Chuyển tiếp
Xong GĐ5 → hoàn thành lộ trình. Xem lại checklist "hoàn thành" ở [README.md](../../../README.md).
