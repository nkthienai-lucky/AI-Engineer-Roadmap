# Giai đoạn 1 — Nền tảng GenAI & Prompting (Tuần 1–3)

> **Mục tiêu:** Hiểu GenAI là gì, gọi được LLM API thành thạo, viết prompt tốt, deploy demo đầu tiên.
> **Đầu ra:** Chatbot CLI + 1 demo Gradio trên Hugging Face Spaces.

---

## 🎓 Nội dung cần học

### Tuần 1 — Nền tảng kỹ thuật
- Python thành thạo: function, class, async/await cơ bản, virtualenv/uv
- Git: commit, branch, PR
- REST API + JSON: gọi API từ Python (`requests`/`httpx`)
- Kiến trúc 1 lời gọi LLM: `prompt → model → completion`, streaming là gì

### Tuần 2 — GenAI overview + Prompting
- LLM là gì, token là gì, context window, temperature/top-p
- Phân biệt: LLM / chat model / embedding model / VLM
- Prompt engineering: zero-shot, few-shot, chain-of-thought, structured output (JSON mode)
- Ranh giới: hallucination, cutoff kiến thức, không có bộ nhớ dài hạn

### Tuần 3 — Làm việc với model local & API
- Cài **Ollama**, chạy model local (Qwen2.5-1.5B, Llama-3.2-3B)
- API kiểu OpenAI-compatible (`base_url` trỏ về Ollama)
- Xây chatbot CLI có streaming + history
- Deploy demo lên Hugging Face Spaces

---

## 📚 Tài nguyên

| Tài nguyên | Dùng để | Link |
|---|---|---|
| **Generative AI for Everyone** (Andrew Ng) | Tổng quan, không code | [deeplearning.ai/courses/generative-ai-for-everyone](https://www.deeplearning.ai/courses/generative-ai-for-everyone) |
| **Hugging Face LLM Course** (chương 1–3) | NLP + tokenization cơ bản | [huggingface.co/learn/nlp-course](https://huggingface.co/learn/nlp-course) |
| **Prompt Engineering Guide** | Kỹ thuật prompt (tham khảo khi cần) | [promptingguide.ai](https://www.promptingguide.ai/) |
| **Ollama** | Chạy model local | [ollama.com](https://ollama.com/) |
| **Gradio** | UI demo nhanh | [gradio.app](https://www.gradio.app/) |
| **OpenAI API docs** (đọc mục Chat Completion) | Hiểu chuẩn API industry | [platform.openai.com/docs](https://platform.openai.com/docs/) |

---

## 🛠️ Project GĐ1 — "Chatbot CLI + Demo"

**Yêu cầu:**
1. CLI chatbot Python: nhập câu hỏi → in streaming câu trả lời, giữ lịch sử hội thoại
2. Chạy được với cả 2 backend: Ollama local **và** API thật (đổi 1 dòng config)
3. Chế độ "hệ thống" (system prompt) tùy chỉnh được
4. Deploy 1 phiên bản web lên **HF Spaces** (Gradio) — có URL public

**Tiêu chí hoàn thành (DoD):**
- [ ] `python chat.py` chạy được, stream mượt
- [ ] Đổi model/backend không phải sửa code logic
- [ ] Demo URL hoạt động, gửi được cho bạn bè xem
- [ ] README có ảnh screenshot + hướng dẫn chạy

---

## ⚠️ Lỗi thường gặp
- Prompt dài dòng lan man → học cách viết prompt ngắn, có role, có ví dụ output
- Không xử lý streaming → UX tệ, học `stream=True` sớm
- Hard-code API key trong code → dùng env var từ đầu
- Chỉ dùng API cloud, ngại local → phải cài Ollama để hiểu model chạy thế nào (RAM, VRAM, tốc độ)

---

## 🔗 Chuyển tiếp
Xong GĐ1 → bạn đã "dùng được" GenAI. GĐ2 ([02-LLM-INTERNALS](../02-LLM-INTERNALS/README.md)) sẽ trả lời câu hỏi *"bên trong cái API đó là gì?"*
