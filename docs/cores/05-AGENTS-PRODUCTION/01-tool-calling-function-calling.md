# 01 — Function / Tool Calling: cơ chế, JSON schema, error handling

> **Mục tiêu:** Hiểu tường tận cơ chế function/tool calling — cách một model "gọi hàm" mà thực ra chỉ sinh JSON — biết định nghĩa tool bằng JSON schema, thiết kế error handling khi tool fail, và viết được một flow gọi tool hoàn chỉnh bằng Python.
>
> Nguồn mở rộng từ: [05-AGENTS-PRODUCTION](README.md)

---

## Function calling hoạt động như thế nào?

Điều quan trọng nhất cần hiểu: **model không hề thực thi code**. Model chỉ sinh ra một mẩu JSON mô tả "tôi muốn gọi hàm X với tham số Y". Toàn bộ việc thực thi do **runtime** (phần code của bạn) làm. Chu trình 4 bước:

1. **Bạn khai báo tool** — gửi cho model danh sách tool kèm JSON schema (tên hàm, mô tả, tham số).
2. **Model sinh JSON gọi tool** — khi câu hỏi cần dữ liệu thật (giá, thời tiết, DB), model output một `tool_call`: tên tool + arguments dạng JSON.
3. **Runtime thực thi** — code của bạn parse JSON đó, gọi hàm thật (API, database, file...), thu kết quả.
4. **Trả kết quả vào context** — kết quả được đưa trở lại messages dưới vai trò `tool` (tool result), model đọc nó và sinh câu trả lời cuối bằng ngôn ngữ tự nhiên.

```
User: "Hà Nội hôm nay bao nhiêu độ?"
  → Model sinh: {"name": "get_weather", "arguments": {"city": "Hanoi"}}
  → Runtime gọi API thời tiết thật → "32°C, nắng"
  → Đưa kết quả vào context
  → Model trả lời: "Hà Nội hôm nay 32°C, trời nắng."
```

Vì JSON chỉ là **văn bản**, model có thể sinh JSON sai (thiếu ngoặc, sai kiểu dữ liệu) — đây là lý do error handling (mục dưới) là bắt buộc, không phải optional.

## JSON schema — cách định nghĩa tool

Mỗi tool gồm: **tên** (snake_case, mô tả hành động), **mô tả** (rất quan trọng — model chọn tool dựa vào đây), và **parameters** là JSON schema chuẩn. Ví dụ:

```json
{
  "name": "get_weather",
  "description": "Lấy thời tiết hiện tại của một thành phố. Dùng khi người dùng hỏi về thời tiết, nhiệt độ, mưa/nắng.",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {"type": "string", "description": "Tên thành phố, ví dụ: Hanoi, Da Nang"},
      "unit": {"type": "string", "enum": ["c", "f"], "description": "Đơn vị nhiệt độ, mặc định c"}
    },
    "required": ["city"]
  }
}
```

Nguyên tắc thiết kế schema tốt:

- **Mô tả tool phải nói "khi nào dùng"**, không chỉ "làm gì" — model phân biệt `get_weather` vs `get_forecast` dựa vào mô tả.
- **Đánh dấu `required` rõ ràng**; tham số optional thì ghi giá trị mặc định trong `description`.
- **Dùng `enum` để đóng khung** giá trị hợp lệ (đơn vị, ngôn ngữ, mode...).
- **Ít tool, mô tả sắc nét** — mỗi tool thừa là một lựa chọn khiến model dễ nhầm (xem file 07 — "ảo tưởng quyền lực").

## Error handling khi tool fail

Tool fail là chuyện **bình thường**: API timeout, tham số sai, DB chết. Sai lầm phổ biến là nuốt lỗi rồi trả `None` — model sẽ bịa kết quả. Quy tắc: **lỗi phải được trả về cho model dưới dạng kết quả tool**, để model tự sửa hướng đi.

```python
def call_tool(name: str, args: dict) -> str:
    try:
        tool = TOOLS[name]
        return tool(**args)                      # JSONDecodeError, KeyError... đều bắt được
    except KeyError:
        return f"ERROR: không có tool tên '{name}'. Danh sách: {list(TOOLS)}"
    except (json.JSONDecodeError, TypeError) as e:
        return f"ERROR: tham số không hợp lệ: {e}. Hãy sinh lại arguments đúng schema."
    except Exception as e:
        return f"ERROR: tool '{name}' thất bại: {e}. Có thể thử lại hoặc báo người dùng."
```

Ba tầng xử lý:

| Loại lỗi | Nguyên nhân | Xử lý |
|---|---|---|
| Sai tên tool | Model "bịa" tên hàm | Trả về danh sách tool thật, cho model chọn lại |
| Sai arguments | JSON sai / thiếu required | Báo lỗi kèm schema, cho model sinh lại |
| Tool thật sự fail | API/DB/network | Trả thông báo lỗi, cho model thử lại hoặc thông báo người dùng |

Bổ sung thêm: **retry có giới hạn** (1–2 lần, không lặp vô hạn), **timeout** cho mọi lời gọi mạng, và **validate JSON bằng schema** (ví dụ `jsonschema` library) trước khi thực thi — chặn input độc hại lẫn lỗi ngớ ngẩn.

## Flow gọi tool hoàn chỉnh (Python tối giản)

```python
messages = [{"role": "user", "content": "Hà Nội hôm nay bao nhiêu độ?"}]

while True:
    resp = llm.chat(messages, tools=TOOLS_SCHEMA)
    if not resp.tool_calls:                       # model trả lời xong → thoát
        print(resp.content); break
    for tc in resp.tool_calls:
        messages.append({"role": "assistant", "tool_calls": [tc]})
        result = call_tool(tc.name, tc.arguments)  # thực thi + error handling ở trên
        messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})
    # vòng lặp quay lại: model đọc kết quả tool rồi quyết định tiếp
```

Điểm cần nhớ: mỗi kết quả tool **phải gắn đúng `tool_call_id`** để model biết kết quả thuộc lời gọi nào (một vòng có thể gọi nhiều tool song song).

## Liên kết

- [02 — Agent loop: ReAct & Planner](02-agent-loop-react-planner.md) — vòng lặp nhiều bước được xây trên chính cơ chế tool calling này
- [04 — Guardrails & An toàn](04-guardrails-and-safety.md) — validate arguments/output của tool trước khi dùng
- [05 — Observability & Eval](05-observability-eval-production.md) — trace từng lần gọi tool để gỡ lỗi
