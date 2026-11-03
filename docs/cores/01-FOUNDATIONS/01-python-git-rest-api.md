# Tuần 1 — Python, Git, REST API & JSON

> **Mục tiêu:** Nắm vững công cụ kỹ thuật cơ bản trước khi chạm vào GenAI: viết Python thành thạo (function, class, async/await, virtualenv/uv), dùng Git để quản lý code (commit, branch, PR), hiểu REST/JSON và gọi được API từ Python bằng `requests`/`httpx`.
> **Đầu ra:** Gọi được một API bất kỳ từ Python, in kết quả JSON; quản lý code bằng Git không sợ mất code.

> Nguồn mở rộng từ: [01-FOUNDATIONS.md](../../../01-FOUNDATIONS.md)

---

## Python thành thạo

Mục tiêu của tuần này không phải học "toàn bộ Python" mà là viết được code sạch, tổ chức được project, và quan trọng nhất: **gọi được LLM API** — vốn chỉ là các hàm Python gửi HTTP request.

### Function — hàm là đơn vị code tái sử dụng

Hàm giúp bạn gói một việc vào một chỗ, gọi lại nhiều lần. Khi làm chatbot, mỗi "việc" (gửi prompt, in streaming, lưu lịch sử) nên là một hàm riêng.

```python
def ask_llm(prompt: str, system: str = "Bạn là trợ lý hữu ích.") -> str:
    """Gửi prompt tới LLM, trả về câu trả lời dạng text."""
    messages = [
        {"role": "system", "content": system},
        {"role": "user", "content": prompt},
    ]
    return call_api(messages)  # giả lập hàm gọi API ở dưới
```

Kiểu dữ liệu trong dấu `-> str` và `prompt: str` là **type hint** — không bắt buộc nhưng nên dùng: giúp bạn (và VS Code) bắt lỗi trước khi chạy.

### Class — gom dữ liệu + hành vi liên quan

Class hữu ích khi bạn cần "đối tượng" giữ trạng thái: ví dụ một phiên chat đang mở (lịch sử tin nhắn + cấu hình model).

```python
class ChatSession:
    def __init__(self, model: str = "qwen2.5:1.5b"):
        self.model = model
        self.history: list[dict] = []

    def ask(self, prompt: str) -> str:
        self.history.append({"role": "user", "content": prompt})
        answer = call_api(self.history, model=self.model)
        self.history.append({"role": "assistant", "content": answer})
        return answer
```

So với hàm rời rạc, class ở đây có lợi: lịch sử hội thoại "đi theo" phiên chat mà không phải truyền tay biến `history` khắp nơi.

### async/await cơ bản — và khi nào dùng

**Ý tưởng:** `async/await` cho phép hàm "chờ" tác vụ chậm (mạng, API) mà **không chặn** chương trình — thay vì đứng chờ response, event loop chạy việc khác.

**Khi nào dùng:** khi bạn cần chạy **nhiều lời gọi API đồng thời** (ví dụ hỏi 3 model cùng lúc), hoặc build web server xử lý nhiều user. Với chatbot CLI đơn giản, code **đồng bộ** là đủ và dễ hiểu hơn.

```python
import asyncio
import httpx

async def fetch_answer(client: httpx.AsyncClient, question: str) -> str:
    resp = await client.post(URL, json={"prompt": question})
    return resp.json()["answer"]

async def main():
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(
            fetch_answer(client, "Trời là gì?"),
            fetch_answer(client, "Biển là gì?"),
        )
    print(results)  # 2 câu trả lời về gần như cùng lúc
```

Nếu viết đồng bộ tuần tự, tổng thời gian = thời gian API 1 + API 2. Với `asyncio.gather`, tổng ≈ max(API 1, API 2).

### virtualenv / uv — môi trường ảo

**Vấn đề:** Project A cần `httpx==0.27`, project B cần `httpx==0.24` — cài chung một chỗ sẽ xung đột.

**virtualenv** tạo thư mục môi trường ảo riêng cho từng project:

```bash
python -m venv .venv        # tạo môi trường ảo
.venv\Scripts\activate      # Windows (bash/zsh: source .venv/bin/activate)
pip install httpx
pip freeze > requirements.txt
```

**uv** là công cụ thay thế nhanh hơn (viết bằng Rust), kiêm nhiệm cả venv + cài package + quản lý Python version:

```bash
uv venv                     # tạo .venv (nhanh hơn venv nhiều)
uv pip install httpx gradio
uv pip freeze > requirements.txt
```

Quy tắc: **mỗi project một `.venv`**, không cài package "global", luôn commit `requirements.txt` để người khác chạy được.

---

## Git: commit, branch, PR

Git giúp bạn lưu "ảnh chụp" (snapshot) của code theo thời gian, quay lại được khi hỏng, và làm việc nhóm không dẫm chân nhau.

### Commit — chụp ảnh code

```bash
git init                      # lần đầu: khởi tạo repo
git add chat.py               # "đóng gói" thay đổi vào staging
git commit -m "feat: add streaming chat loop"
```

Nguyên tắc commit tốt: **một commit = một việc**, message mô tả *tại sao*/*làm gì* (không phải "update", "fix"). Prefix thường dùng: `feat:`, `fix:`, `docs:`, `refactor:`.

### Branch — nhánh làm song song

```bash
git checkout -b feature/gradio-demo   # tạo + chuyển sang nhánh mới
# ...code, commit...
```

Nhánh `main` luôn chạy được; tính năng mới phát triển trên nhánh riêng. Hỏng? Xóa nhánh, code gốc không sao.

### PR (Pull Request) — đề nghị hợp nhất

Khi xong việc trên nhánh, bạn mở PR để người khác **review code** trước khi merge vào `main`. Trên GitHub/GitLab, PR là nơi thảo luận, chạy CI (test tự động), và ghi lại *tại sao* thay đổi này tồn tại. Dù làm solo, thói quen "nhánh → PR → merge" giữ lịch sử sạch và là quy trình chuẩn ở công ty.

---

## REST API + JSON

### JSON là gì

JSON (JavaScript Object Notation) là định dạng dữ liệu text phổ biến nhất để máy móc trao đổi: object `{}` với cặp `key: value`, mảng `[]`. Python dict có cấu trúc gần giống JSON:

```python
{"role": "user", "content": "Xin chào", "temperature": 0.7}   # JSON
{"role": "user", "content": "Xin chào", "temperature": 0.7}   # Python dict (giống hệt)
```

Đổi qua lại: `json.dumps(dict)` → chuỗi JSON, `json.loads(chuỗi)` → dict.

### REST là gì

REST (Representational State Transfer) là **quy ước thiết kế API qua HTTP**: mỗi "tài nguyên" (user, tin nhắn, model) có một URL, và thao tác bằng HTTP method:

| Method | Ý nghĩa | Ví dụ |
|---|---|---|
| `GET` | Lấy dữ liệu | `GET /models` — xem danh sách model |
| `POST` | Tạo/tác động | `POST /chat` — gửi câu hỏi, nhận trả lời |
| `PUT`/`PATCH` | Cập nhật | `PATCH /chat/123` — sửa phiên chat |
| `DELETE` | Xóa | `DELETE /chat/123` — xóa phiên chat |

Gọi API "kiểu REST" nghĩa là: gửi HTTP request với method + URL + body JSON, nhận về response JSON kèm mã trạng thái (`200` OK, `401` sai key, `429` quá tải...). **LLM API mà bạn sẽ dùng chính là REST API** — vì vậy hiểu REST ở đây là học trực tiếp cho việc gọi model.

### Gọi API từ Python: requests / httpx

`requests` là thư viện kinh điển (đồng bộ), `httpx` là bản hiện đại hơn (đồng bộ **và** async, hỗ trợ streaming tốt — hợp với LLM API).

```python
import requests

resp = requests.post(
    "https://api.example.com/v1/chat",
    headers={"Authorization": "Bearer sk-xxx"},
    json={"message": "Xin chào"},
    timeout=30,
)
resp.raise_for_status()          # báo lỗi sớm nếu 4xx/5xx
data = resp.json()
print(data["reply"])
```

Với `httpx` async (cho nhiều lời gọi đồng thời):

```python
import httpx

async def main():
    async with httpx.AsyncClient(timeout=30) as client:
        r = await client.post(URL, json={"message": "Xin chào"})
        print(r.json()["reply"])
```

### Kiến trúc 1 lời gọi LLM

Mọi tương tác LLM đều theo mô hình: **`prompt → model → completion`** — bạn gửi prompt (danh sách message), server chạy model, trả về completion (text sinh ra). **Streaming** nghĩa là completion không đến trọn một cục mà đến từng mảnh (token) khi model vừa sinh ra — chi tiết hơn ở file [04-local-models-ollama-api.md](04-local-models-ollama-api.md).

---

## Liên kết

- [02-genai-concepts-token-context.md](02-genai-concepts-token-context.md) — token, context window, các loại model (áp dụng trực tiếp kiến thức API ở đây)
- [04-local-models-ollama-api.md](04-local-models-ollama-api.md) — dùng `httpx`/`requests` để gọi Ollama và API thật
- [05-project-chatbot-cli.md](05-project-chatbot-cli.md) — project tuần 1–3 dùng toàn bộ kỹ năng trong file này
