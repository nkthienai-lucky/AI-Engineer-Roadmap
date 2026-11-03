# Tài nguyên học tập & Lỗi thường gặp (GĐ1)

> **Mục tiêu:** Biết dùng đúng 6 tài nguyên cốt lõi của GĐ1 (mỗi tài liệu để làm gì, đọc khi nào), và hiểu sâu 4 lỗi thường gặp của người mới — *vì sao* lỗi đó xảy ra và cách tránh cụ thể.
> **Đầu ra:** Lộ trình đọc tài liệu không lan man + checklist phòng lỗi trước khi nộp project GĐ1.

> Nguồn mở rộng từ: [01-FOUNDATIONS.md](../../../01-FOUNDATIONS.md)

---

## Bảng tài nguyên

Giữ nguyên bảng gốc từ nguồn, kèm cột giải thích mở rộng *dùng tài liệu đó vào việc gì trong GĐ1*:

| Tài nguyên | Dùng để | Link | Đọc gì cụ thể / dùng khi nào |
|---|---|---|---|
| **Generative AI for Everyone** (Andrew Ng) | Tổng quan, không code | [deeplearning.ai/courses/generative-ai-for-everyone](https://www.deeplearning.ai/courses/generative-ai-for-everyone) | Xem trước khi code: hiểu GenAI giải quyết được gì, hạn chế ra sao (hallucination, cutoff — [02-genai-concepts-token-context.md](02-genai-concepts-token-context.md)). Không cần cài gì, hợp cho buổi tối đầu tiên. |
| **Hugging Face LLM Course** (chương 1–3) | NLP + tokenization cơ bản | [huggingface.co/learn/nlp-course](https://huggingface.co/learn/nlp-course) | Chương 1–3 đúng phần cần cho GĐ1: token là gì, tokenizer cắt text ra sao, pipeline cơ bản. Học song song Tuần 2 để hiểu phần "token ≠ từ" có ví dụ chạy được. |
| **Prompt Engineering Guide** | Kỹ thuật prompt (tham khảo khi cần) | [promptingguide.ai](https://www.promptingguide.ai/) | Không đọc hết từ đầu đến cuối — tra cứu theo nhu cầu: mục zero-shot/few-shot/chain-of-thought khi làm [03-prompt-engineering.md](03-prompt-engineering.md), mục structured output khi cần JSON mode. |
| **Ollama** | Chạy model local | [ollama.com](https://ollama.com/) | Cài đặt + danh sách model để kéo về (Qwen2.5-1.5B, Llama-3.2-3B). Làm theo [04-local-models-ollama-api.md](04-local-models-ollama-api.md) ở Tuần 3. |
| **Gradio** | UI demo nhanh | [gradio.app](https://www.gradio.app/) | Tài liệu `ChatInterface` để dựng demo web cho project GĐ1 ([05-project-chatbot-cli.md](05-project-chatbot-cli.md)). Chỉ cần mục Quickstart là đủ cho demo. |
| **OpenAI API docs** (đọc mục Chat Completion) | Hiểu chuẩn API industry | [platform.openai.com/docs](https://platform.openai.com/docs/) | Đọc mục **Chat Completion**: cấu trúc `messages`, tham số `temperature`/`top_p`, ý nghĩa `stream=True`. Đây là "chuẩn" mà Ollama cũng tương thích — hiểu nó là hiểu mọi LLM API. |

**Thứ tự đọc gợi ý (3 tuần):** Tuần 1 — Andrew Ng (tổng quan) + docs REST/HTTP trong lúc code; Tuần 2 — HF LLM Course ch.1–3 + Prompt Engineering Guide tra cứu; Tuần 3 — Ollama docs + OpenAI Chat Completion + Gradio Quickstart.

### Cách dùng tài liệu hiệu quả (không đọc lan man)

Với 6 tài nguyên trên, ba thói quen sau giúp bạn không chìm trong tài liệu:

- **Đọc theo project, không đọc theo sách:** mở tài liệu khi project cần — ví dụ đang làm streaming thì đọc mục `stream` trong OpenAI API docs, không đọc hết docs trước khi code.
- **Tài liệu tham khảo ≠ tài liệu học thuộc:** Prompt Engineering Guide hàng trăm trang — chỉ tra cứu kỹ thuật đang cần; Andrew Ng mới là tài liệu "học" theo tuần.
- **Vừa đọc vừa chạy:** mọi ví dụ trong docs đều chạy được ngay với Ollama local (miễn phí) — đọc mà không chạy thì 1 tuần sau sẽ quên hết.

```python
# Thói quen nhỏ: mỗi kỹ thuật học được, gói thành hàm riêng để tái sử dụng
# ví dụ sau khi đọc mục JSON mode trong OpenAI API docs:
def ask_json(system: str, user: str) -> dict:
    import json
    text = call_llm(system=system, user=user, json_mode=True)
    return json.loads(text)
```

---

## 4 lỗi thường gặp — mở rộng: vì sao xảy ra & cách tránh

### 1. Prompt dài dòng lan man

- **Biểu hiện:** Prompt hàng trăm chữ, trộn lẫn vai trò, ngữ cảnh, 5–6 yêu cầu một lượt → model bỏ sót yêu cầu, output lan man theo prompt.
- **Vì sao xảy ra:** Người mới tưởng "viết càng đầy đủ model càng hiểu" — thực tế mỗi câu thừa là nhiễu, và nhiều yêu cầu trong một prompt khiến model ưu tiên nhầm chỗ.
- **Cách tránh cụ thể:** Áp dụng 3 nguyên tắc từ nguồn — *ngắn gọn, có role, có ví dụ output*. Viết prompt xong, đọc lại và cắt câu nào không thay đổi được output. Tách yêu cầu phức tạp thành nhiều lượt thay vì nhồi một prompt. Tham khảo kỹ thuật trong [03-prompt-engineering.md](03-prompt-engineering.md).

### 2. Không xử lý streaming → UX tệ

- **Biểu hiện:** App "đứng im" 10–30 giây rồi hiện trọn câu trả lời — user tưởng bị đơ/thoát ra trước khi đọc được gì.
- **Vì sao xảy ra:** Người mới quen model trả kết quả như hàm bình thường (input → output trọn vẹn), không biết API mặc định trả "một cục" và `stream=True` đổi hoàn toàn cảm giác sử dụng.
- **Cách tránh cụ thể:** Bật `stream=True` và in/hiển thị **từng chunk ngay khi nhận** (`print(delta, end="", flush=True)` trong CLI; yield token trong Gradio). Học streaming ngay từ version đầu của chatbot, đừng để "xong rồi bổ sung" — snippet đầy đủ ở [04-local-models-ollama-api.md](04-local-models-ollama-api.md).

### 3. Hard-code API key trong code

- **Biểu hiện:** `api_key="sk-abc..."` nằm thẳng trong file `.py`, vô tình commit lên GitHub → key bị bot quét và dùng chùa.
- **Vì sao xảy ra:** Khi mới học, hard-code là cách nhanh nhất để code chạy được — và người mới không biết key là "bí mật" có giá trị tiền thật, GitHub public thì ai cũng thấy.
- **Cách tránh cụ thể:** Dùng env var **từ dòng code đầu tiên** (`os.getenv("LLM_API_KEY")`), key để trong terminal session hoặc file `.env` nằm trong `.gitignore`. Quen tay với snippet trong [04-local-models-ollama-api.md](04-local-models-ollama-api.md); nếu lỡ commit key → thu hồi/đổi key ngay trên trang nhà cung cấp.

Ví dụ kiểm tra nhanh trước khi commit:

```bash
git diff | grep -i "sk-"     # không thấy gì là an toàn
```

### 4. Chỉ dùng API cloud, ngại local

- **Biểu hiện:** Không bao giờ cài Ollama, mọi thứ chạy qua API trả phí → không hình dung được model "nặng" ra sao, đổi máy là không chạy được.
- **Vì sao xảy ra:** Cài local có vẻ "nặng", mất thời gian, trong khi API cloud chỉ cần 1 dòng lệnh. Nhưng chính vì thế người mới bỏ lỡ hiểu biết về RAM/VRAM/tốc độ sinh token — nền tảng cho GĐ2 "bên trong API là gì".
- **Cách tránh cụ thể:** Bắt buộc cài Ollama và chạy Qwen2.5-1.5B (model nhẹ, laptop thường vẫn chạy được) theo [04-local-models-ollama-api.md](04-local-models-ollama-api.md). Quan sát RAM bị chiếm và token/s khi sinh — hiểu *vì sao* model lớn cần phần cứng mạnh, và tận dụng backend local để test prompt miễn phí.

Một buổi tối với Ollama là đủ: kéo model, chạy vài prompt, mở Task Manager để thấy RAM thực tế bị chiếm — bài học trực quan hơn mọi con số trong tài liệu.

---

## Tóm tắt checklist phòng lỗi

Trước khi nộp project GĐ1, tự hỏi:

- [ ] Prompt đã ngắn gọn, có role, có ví dụ output mẫu chưa? (lỗi #1)
- [ ] Chatbot đã stream từng token, chưa "đứng im" không? (lỗi #2)
- [ ] Trong code có chỗ nào chứa key/thông tin bí mật không? (lỗi #3)
- [ ] Đã chạy thử backend Ollama local ít nhất một lần chưa? (lỗi #4)

---

## Liên kết

- [03-prompt-engineering.md](03-prompt-engineering.md) — sửa lỗi #1 (prompt lan man)
- [04-local-models-ollama-api.md](04-local-models-ollama-api.md) — sửa lỗi #2 (streaming), #3 (env var), #4 (Ollama local)
- [05-project-chatbot-cli.md](05-project-chatbot-cli.md) — checklist nghiệm thu project GĐ1 trước khi chuyển GĐ2
