# 03 — Memory (ngắn hạn vs dài hạn) & MCP

> **Mục tiêu:** Phân biệt memory ngắn hạn (context window) và memory dài hạn (vector store), hiểu cơ chế lưu/truy xuất của từng loại, và nắm khái niệm **MCP (Model Context Protocol)** — chuẩn kết nối tool mới mà mọi agent cần biết.
>
> Nguồn mở rộng từ: [05-AGENTS-PRODUCTION](README.md)

---

## So sánh nhanh hai loại memory

| | Ngắn hạn (context window) | Dài hạn (vector store) |
|---|---|---|
| Nơi sống | Trong prompt mỗi lượt gọi | Ngoài đĩa / vector DB |
| Thời hạn | Một phiên hội thoại | Vô hạn (cho đến khi xóa) |
| Dung lượng | Giới hạn bởi context window | Gần như không giới hạn |
| Truy xuất | Có sẵn trong `messages` | Phải embedding + vector search |
| Chi phí/token | Tốn token **mọi lượt** nếu không cắt | Chỉ tốn khi truy xuất top-k |
| Phù hợp | Ngữ cảnh hội thoại tức thời | Sở thích, kiến thức, lịch sử dài |

Thiết kế tốt là **kết hợp cả hai**: context window giữ cửa sổ trượt ngắn hạn, còn thông tin bền được ghi ra vector store rồi truy xuất đúng lúc vào context khi cần — "đúng thông tin, đúng lúc" chính là chủ đề của khóa Context Engineering (link ở [file 07](07-resources-and-common-pitfalls.md)).

## Memory ngắn hạn — context window

Memory ngắn hạn **chính là context window**: toàn bộ messages (system prompt + lịch sử hội thoại + kết quả tool) mà model nhìn thấy trong lượt gọi hiện tại. "Ghi nhớ" ở đây nghĩa là *đưa trở lại vào prompt*.

Cơ chế: mỗi lượt hội thoại, lịch sử được nối vào `messages` rồi gửi cùng request. Điểm yếu cốt lõi: **context window có giới hạn** (vài chục nghìn token với model nhỏ chạy local ≤3B, như stack trên Oracle VPS) và **token trong context tốn tiền/điện mọi lượt** — lịch sử 50 lượt không cắt sẽ phình nhanh.

Chiến lược giữ context gọn (context engineering — xem tài nguyên DeepLearning.AI ở [file 07](07-resources-and-common-pitfalls.md)):

- **Cửa sổ trượt**: giữ N lượt gần nhất, bỏ dần lượt cũ.
- **Tóm tắt (summarize)**: thay 10 lượt cũ bằng 1 đoạn tóm tắt do chính LLM sinh.
- **Ưu tiên đưa lại**: chỉ đưa vào lại những gì liên quan câu hỏi hiện tại (dùng retrieval — nối với memory dài hạn dưới đây).

```python
# Cửa sổ trượt + tóm tắt đơn giản
if count_tokens(messages) > MAX_CTX:
    old, recent = messages[1:-6], messages[-6:]
    summary = llm("Tóm tắt hội thoại sau trong 5 câu: " + flatten(old))
    messages = [messages[0], {"role": "system", "content": "Tóm tắt trước đó: " + summary}] + recent
```

## Memory dài hạn — vector store

Memory dài hạn sống **ngoài** context window: ghi xuống đĩa (thường là vector store), khi cần thì **truy xuất** đúng mẩu liên quan rồi nhét vào context. Đây chính là kỹ thuật RAG đã học ở GĐ3 — áp dụng cho "ký ức của agent" thay vì tài liệu.

Cơ chế lưu/truy xuất:

1. **Lưu**: mỗi mẩu thông tin cần nhớ (sở thích người dùng, kết quả bước trước, quyết định quan trọng) → embedding → ghi vào vector DB kèm metadata (user_id, thời gian, loại).
2. **Truy xuất**: khi có câu hỏi mới → embedding câu hỏi → **vector search** lấy top-k mẩu gần nhất → đưa vào prompt như "ký ức liên quan".

```python
# Lưu
memory_db.add(embed(user_msg), text=user_msg, meta={"user_id": uid, "ts": now()})
# Truy xuất trước mỗi lượt trả lời
memories = memory_db.search(embed(query), top_k=3, filter={"user_id": uid})
context += "\n".join(m.text for m in memories)
```

Hai lưu ý quan trọng: (a) **lọc theo `user_id`** — ký ức của người A không được rò sang người B; (b) **không phải thứ gì cũng lưu** — chỉ lưu thông tin bền (sở thích, quyết định), không lưu rác hội thoại, vì index phình làm retrieval kém và tốn phí.

## MCP — Model Context Protocol

MCP là **chuẩn kết nối tool mới** — đọc khái niệm ở mức này là đủ cho giai đoạn học. Trước MCP, mỗi ứng dụng tự viết adapter riêng cho từng tool/mỗi LLM (N ứng dụng × M tool = N×M đoạn code kết nối). MCP chuẩn hoá đầu này thành **một giao thức chung**: tool/data source đóng gói thành **MCP server** expose các "tool" theo chuẩn, mọi ứng dụng hỗ trợ MCP (**MCP client**) cắm vào là dùng được ngay — N×M còn N+M.

Cấu trúc: **MCP server** (bên cung cấp công cụ: database, GitHub, file system...) expose tool/resources theo chuẩn; **MCP client** (agent/ứng dụng LLM) phát hiện và gọi chúng giống như tool nội bộ. Lợi ích: dùng lại tool cộng đồng thay vì tự viết, đổi model/framework không phải viết lại kết nối tool. Trong lộ trình này chỉ cần **đọc khái niệm** — chưa cần tự viết MCP server; khi tự build agent production thì đây là đường để tích hợp tool bên ngoài.

Ba khái niệm nên nhớ khi đọc tài liệu MCP:

- **Tool** — hành động model gọi được (query database, gửi email...) — giống tool trong function calling (file 01).
- **Resource** — dữ liệu đọc được (nội dung file, bảng DB) — MCP server expose, client kéo vào context.
- **Prompt / template** — prompt mẫu do server cung cấp sẵn để client dùng lại.

Ví dụ trực quan: thay vì tự viết 3 đoạn code kết nối Postgres, GitHub, Google Drive cho agent, bạn cắm 3 MCP server có sẵn — agent của bạn "thấy" ngay các tool `query_db`, `create_issue`, `search_drive` và gọi qua cơ chế tool calling bình thường.

## Liên kết

- [01 — Function/Tool Calling](01-tool-calling-function-calling.md) — cách agent gọi tool mà MCP chuẩn hoá
- [04 — Guardrails & An toàn](04-guardrails-and-safety.md) — dữ liệu truy xuất từ memory/RAG có thể chứa prompt injection
- [06 — Production hóa & Capstone](06-productionization-and-capstone.md) — memory + MCP nằm trong stack capstone
