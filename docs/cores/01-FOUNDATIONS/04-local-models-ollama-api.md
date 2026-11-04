# Tuần 3 — Model local với Ollama & Gọi API kiểu OpenAI-compatible

> **Mục tiêu:** Cài và chạy được model local bằng Ollama (Qwen2.5-1.5B, Llama-3.2-3B), gọi model qua API kiểu OpenAI-compatible với `base_url` trỏ về Ollama (đổi 1 dòng config để chuyển backend), hiểu streaming (`stream=True`) và tại sao nó bắt buộc với UX, dùng env var cho API key.
> **Đầu ra:** Chạy được cùng một đoạn code với cả model local lẫn API thật, chỉ khác một dòng cấu hình.

> Nguồn mở rộng từ: [01-FOUNDATIONS](README.md)

---

## Ollama là gì

Ollama ([ollama.com](https://ollama.com/)) là công cụ giúp **tải và chạy model LLM ngay trên máy bạn** — không cần GPU xịn (model nhỏ chạy được trên CPU), không cần tài khoản, không tốn tiền API. Nó đóng gói model thành dạng "dễ kéo về" và expose sẵn một HTTP server local (mặc định `http://localhost:11434`).

Tại sao phải biết chạy local (thay vì chỉ dùng API cloud):

- **Hiểu model chạy thế nào:** trực tiếp thấy RAM/VRAM bị chiếm, tốc độ sinh token (token/s) phụ thuộc phần cứng ra sao — nền tảng cho GĐ2 về internal.
- **Chủ động & riêng tư:** data không rời máy, không phụ thuộc mạng/key/quota.
- **Miễn phí:** thoải mái thử prompt, thử model, không lo hóa đơn.

## Cài và chạy model local

Cài Ollama theo hướng dẫn tại [ollama.com](https://ollama.com/), sau đó kéo và chạy model từ terminal:

```bash
ollama run qwen3.5:9b         # model 6.6GB — LLM chính của roadmap (xem ENVIRONMENT-SETUP.md)
ollama run qwen2.5:1.5b       # model ~1GB — bản siêu nhẹ, chạy mượt cả trên CPU/laptop
```

Hai model nên bắt đầu:

| Model | Cỡ | Đặc điểm |
|---|---|---|
| **Qwen3.5-9B** | ~9 tỷ tham số | LLM chính của roadmap — thông minh nhất nhóm dense <10B, tiếng Việt tốt |
| **Qwen2.5-1.5B** | ~1.5 tỷ tham số | Rất nhẹ, nhanh, đủ cho chatbot demo và test pipeline trên máy yếu |

Lưu ý thực tế: "1.5B/9B" là số tham số — càng lớn thì càng **tốn RAM/VRAM và sinh chậm hơn**, nhưng thông minh hơn. Với máy 8GB VRAM như của roadmap, Qwen3.5-9B (6.6GB Q4) là lựa chọn chính; máy yếu thì bắt đầu bằng bản 1.5B. Sau khi `ollama run`, server API local đã sẵn sàng ở `localhost:11434`.

## API kiểu OpenAI-compatible — "đổi 1 dòng config"

OpenAI API là **chuẩn de-facto** của industry: một endpoint `/v1/chat/completions`, nhận danh sách `messages`, trả về completion. Ollama (và nhiều nhà cung cấp khác) cài sẵn API **tương thích chuẩn đó** — nghĩa là code viết cho OpenAI chạy được với Ollama nếu chỉ đổi `base_url`:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",   # ← DÒNG CONFIG DUY NHẤT cần đổi
    api_key="ollama",                        # local không kiểm key, để giá trị bất kỳ
)
resp = client.chat.completions.create(
    model="qwen3.5:9b",
    messages=[{"role": "user", "content": "Xin chào!"}],
)
print(resp.choices[0].message.content)
```

Đổi sang API thật: sửa `base_url` thành endpoint của nhà cung cấp (ví dụ `https://api.openai.com/v1`) và lấy API key thật — **code logic còn lại giữ nguyên**. Đây chính là cơ sở của yêu cầu 2 trong project GĐ1 ("chạy được cả 2 backend, đổi 1 dòng config"): gom `base_url`, `model`, `api_key` vào một module config riêng ([05-project-chatbot-cli.md](05-project-chatbot-cli.md)), phần còn lại của app không biết mình đang nói chuyện với model nào.

## Streaming (`stream=True`) — và tại sao bắt buộc với UX

Mặc định, API trả kết quả **trọn một cục** sau khi model sinh xong toàn bộ — với câu trả lời dài, user phải chờ 5–30 giây nhìn màn hình trống, không biết model "có sống hay không". **Streaming** là server trả từng token ngay khi model vừa sinh ra:

```python
stream = client.chat.completions.create(
    model="qwen3.5:9b",
    messages=[{"role": "user", "content": "Kể 1 câu chuyện ngắn"}],
    stream=True,
)
for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)   # in ngay từng mảnh
```

Tại sao gọi là **bắt buộc với UX**:

- **Phản hồi tức thì:** user thấy chữ xuất hiện trong ~1 giây đầu, cảm giác "nhanh" dù tổng thời gian sinh không đổi (tâm lý người dùng: chờ có nội dung để đọc ≈ không phải chờ).
- **Biết hệ thống còn sống:** token nhấp nháy liên tục thay vì màn hình treo — giảm cảm giác bị đơ/lỗi.
- **Dừng được sớm:** user đọc đủ thì hủy, không tốn token sinh phần thừa.

Nguyên tắc từ nguồn: *"Không xử lý streaming → UX tệ, học `stream=True` sớm"* — hãy đưa streaming vào chatbot từ phiên bản đầu tiên, đừng để sau mới "bổ sung".

## Env var cho API key — không bao giờ hard-code

**Vấn đề:** viết thẳng key vào code (`api_key="sk-abc123..."`) rồi commit lên GitHub → bot quét key trong vài phút, key bị dùng chùa, bạn bị tính tiền. Đây là tai nạn rất phổ biến với người mới.

**Giải pháp:** để key trong **biến môi trường** (env var), đọc lúc chạy:

```python
import os
from openai import OpenAI

client = OpenAI(
    base_url=os.getenv("LLM_BASE_URL", "http://localhost:11434/v1"),
    api_key=os.getenv("LLM_API_KEY", "ollama"),   # key đọc từ env, không nằm trong code
)
```

```bash
# Windows PowerShell
$env:LLM_API_KEY = "sk-..."
# Linux/macOS
export LLM_API_KEY="sk-..."
```

Lợi ích: code commit thoải mái (không chứa bí mật), đổi key/đổi backend không sửa code, `.gitignore` file `.env` nếu muốn lưu key trong file cục bộ. Đây chính là lỗi thường gặp thứ 3 trong nguồn gốc: *"Hard-code API key trong code → dùng env var từ đầu"*.

---

## Liên kết

- [01-python-git-rest-api.md](01-python-git-rest-api.md) — REST/JSON: nền tảng để hiểu endpoint `/v1/chat/completions`
- [05-project-chatbot-cli.md](05-project-chatbot-cli.md) — module config backend + streaming trong project GĐ1
- [06-resources-and-common-pitfalls.md](06-resources-and-common-pitfalls.md) — tài nguyên Ollama và các lỗi thường gặp khi làm quen
