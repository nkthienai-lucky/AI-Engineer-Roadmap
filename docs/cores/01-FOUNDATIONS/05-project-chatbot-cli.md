# Project GĐ1 — "Chatbot CLI + Demo"

> **Mục tiêu:** Spec đầy đủ cho project giai đoạn 1: 4 yêu cầu, tiêu chí hoàn thành (DoD), gợi ý cấu trúc code (module config backend, chat loop, streaming), cách deploy demo Gradio lên Hugging Face Spaces, và checklist nghiệm thu cuối cùng.
> **Đầu ra:** Một chatbot CLI chạy được cả backend local lẫn API thật + 1 demo web public trên HF Spaces.

> Nguồn mở rộng từ: [01-FOUNDATIONS](README.md)

---

## Tổng quan

Project GĐ1 là "bài kiểm tra" rằng bạn đã **dùng được** GenAI: không chỉ gọi API thành công, mà còn đóng gói thành sản phẩm nhỏ có cấu hình sạch, UX streaming mượt, và deploy được cho người khác xem.

---

## 4 yêu cầu

Yêu cầu gốc từ nguồn, kèm giải thích *tại sao yêu cầu đó tồn tại*:

1. **CLI chatbot Python: nhập câu hỏi → in streaming câu trả lời, giữ lịch sử hội thoại.**
   "Streaming" là in từng token khi model sinh (xem [04-local-models-ollama-api.md](04-local-models-ollama-api.md)); "giữ lịch sử" nghĩa là các lượt trước được gửi kèm prompt các lượt sau (bộ nhớ ngắn hạn do **code tự quản lý**, model không tự nhớ — xem [02-genai-concepts-token-context.md](02-genai-concepts-token-context.md)).

2. **Chạy được với cả 2 backend: Ollama local và API thật (đổi 1 dòng config).**
   Nhờ API kiểu OpenAI-compatible, cùng một code gọi model hoạt động với cả hai — chỉ khác `base_url` (và key). Yêu cầu này ép bạn **tách config khỏi logic** ngay từ đầu.

3. **Chế độ "hệ thống" (system prompt) tùy chỉnh được.**
   User sửa được system prompt (qua biến môi trường, file config, hoặc flag `--system`) để đổi vai trò/phong cách chatbot mà không sửa code — ứng dụng trực tiếp phần system prompt trong [03-prompt-engineering.md](03-prompt-engineering.md).

4. **Deploy 1 phiên bản web lên HF Spaces (Gradio) — có URL public.**
   Gradio ([gradio.app](https://www.gradio.app/)) dựng UI chat nhanh với vài dòng code; Hugging Face Spaces host app đó lên web với URL công khai gửi được cho bạn bè.

## Tiêu chí hoàn thành (DoD)

Giữ nguyên từ nguồn:

- [ ] `python chat.py` chạy được, stream mượt
- [ ] Đổi model/backend không phải sửa code logic
- [ ] Demo URL hoạt động, gửi được cho bạn bè xem
- [ ] README có ảnh screenshot + hướng dẫn chạy

Giải thích từng ý: "stream mượt" = token in ra liên tục không bị đơ cục (flush từng mảnh); "không sửa code logic" = chỉ đổi file config/env var, các hàm chat loop không bị đụng vào; "URL hoạt động" = người khác mở link là chat được ngay; README = bộ mặt của project — có ảnh chứng minh demo chạy thật + vài lệnh để người lạ tái hiện được.

---

## Gợi ý cấu trúc code

```
chatbot-cli/
├── chat.py          # entrypoint: chat loop đọc stdin, in streaming
├── config.py        # module config backend (base_url, model, api_key, system)
├── llm.py           # wrapper gọi API (OpenAI-compatible client)
└── README.md
```

### Module config backend (`config.py`) — gom mọi thứ "đổi được"

```python
import os

CONFIG = {
    "base_url": os.getenv("LLM_BASE_URL", "http://localhost:11434/v1"),
    "api_key": os.getenv("LLM_API_KEY", "ollama"),
    "model": os.getenv("LLM_MODEL", "qwen2.5:1.5b"),
    "system": os.getenv("LLM_SYSTEM", "Bạn là trợ lý hữu ích, trả lời bằng tiếng Việt."),
}
```

Đổi backend = đổi env var (`LLM_BASE_URL`), đổi model = đổi `LLM_MODEL` — **không file nào khác bị ảnh hưởng**, thỏa DoD #2.

### Hàm chat loop (`chat.py`) — giữ history, gọi streaming

```python
from config import CONFIG
from llm import stream_chat

def chat_loop():
    history = [{"role": "system", "content": CONFIG["system"]}]
    while True:
        question = input("Bạn: ").strip()
        if question.lower() in {"exit", "quit"}:
            break
        history.append({"role": "user", "content": question})
        print("Bot: ", end="", flush=True)
        answer = stream_chat(history)      # in từng token ngay khi nhận
        print()
        history.append({"role": "assistant", "content": answer})

if __name__ == "__main__":
    chat_loop()
```

### Streaming trong wrapper (`llm.py`)

```python
from openai import OpenAI
from config import CONFIG

client = OpenAI(base_url=CONFIG["base_url"], api_key=CONFIG["api_key"])

def stream_chat(messages: list[dict]) -> str:
    stream = client.chat.completions.create(
        model=CONFIG["model"], messages=messages, stream=True,
    )
    parts = []
    for chunk in stream:
        delta = chunk.choices[0].delta.content
        if delta:
            print(delta, end="", flush=True)   # stream mượt = flush từng mảnh
            parts.append(delta)
    return "".join(parts)
```

---

## Deploy Gradio lên HF Spaces

Các bước (dùng Gradio — [gradio.app](https://www.gradio.app/)):

1. **Viết app Gradio** — UI chat dựng sẵn với `gr.ChatInterface`:

```python
import gradio as gr
from llm import stream_chat

def respond(message, history):          # history dạng [(user, bot), ...]
    messages = [{"role": "system", "content": CONFIG["system"]}]
    for u, b in history:
        messages += [{"role": "user", "content": u}, {"role": "assistant", "content": b}]
    messages.append({"role": "user", "content": message})
    # stream_chat_plain: như stream_chat nhưng trả text thay vì print ra terminal
    return "".join(chunk_text for chunk_text in stream_chat_plain(messages))

demo = gr.ChatInterface(respond, title="Chatbot GĐ1")
demo.launch()
```

2. **Tạo Space trên Hugging Face:** chọn SDK **Gradio**, đặt tên public.
3. **Đẩy code lên Space** (repo HF của Space) với `requirements.txt` (gradio, openai) và `app.py` là entrypoint.
4. **Cấu hình secrets:** đặt `LLM_BASE_URL`, `LLM_API_KEY`, `LLM_MODEL` trong mục Settings → Secrets của Space (API key **không** hard-code — [04-local-models-ollama-api.md](04-local-models-ollama-api.md)). Lưu ý: Ollama local không chạy trên Spaces được → Space phải trỏ `LLM_BASE_URL` tới API thật.

Tách `stream_chat` (in ra terminal, dùng cho CLI) và `stream_chat_plain` (trả text/yield token, dùng cho Gradio) — cùng một wrapper gọi API, khác cách hiển thị; hoặc gộp thành một hàm với tham số `echo=True/False`.
5. Space build xong → có **URL public** gửi cho bạn bè (thỏa yêu cầu 4).

---

## Checklist nghiệm thu

Điểm qua từng mục trước khi tuyên bố xong GĐ1:

- [ ] `python chat.py` chạy được, stream mượt — test bằng câu hỏi dài, xem token có ra từng mảnh không
- [ ] Đổi model/backend không phải sửa code logic — thử đổi env var sang backend còn lại, code chạy nguyên trạng
- [ ] Chế độ system prompt tùy chỉnh — đổi `LLM_SYSTEM` thấy phong cách bot đổi theo
- [ ] Lịch sử hội thoại giữ đúng — hỏi "câu trước tôi hỏi gì?" và bot trả lời được
- [ ] Demo URL hoạt động, gửi được cho bạn bè xem — mở link ở máy khác/chat thử
- [ ] README có ảnh screenshot + hướng dẫn chạy — người lạ clone về chạy được trong 5 phút
- [ ] Không hard-code API key — tìm `sk-` trong code phải không thấy gì

---

## Liên kết

- [01-python-git-rest-api.md](01-python-git-rest-api.md) — kỹ năng Python/Git/REST nền tảng của project này
- [03-prompt-engineering.md](03-prompt-engineering.md) — nội dung cho chế độ system prompt (yêu cầu 3)
- [04-local-models-ollama-api.md](04-local-models-ollama-api.md) — streaming, OpenAI-compatible client, env var (yêu cầu 1–2)
- [06-resources-and-common-pitfalls.md](06-resources-and-common-pitfalls.md) — tài nguyên Gradio và 4 lỗi thường gặp khi làm project
