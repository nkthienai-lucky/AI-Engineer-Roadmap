# 02 — Agent loop: ReAct, Planner vs Executor, LangGraph

> **Mục tiêu:** Hiểu agent loop — vòng lặp "nghĩ → làm → quan sát" cho phép LLM giải quyết tác vụ nhiều bước; phân biệt planner và executor; và biết khi nào nên dùng framework **LangGraph** (agent có trạng thái) thay vì function-calling thuần.
>
> Nguồn mở rộng từ: [05-AGENTS-PRODUCTION](README.md)

---

## Agent loop là gì?

Một lời gọi LLM chỉ trả lời **một lần**. Agent loop là vòng lặp quanh LLM: model sinh hành động (gọi tool), runtime thực thi, đưa kết quả quay lại, model lại quyết định bước tiếp — cho đến khi có câu trả lời cuối hoặc đạt giới hạn số bước. Nền tảng kỹ thuật chính là function calling (file 01), chỉ khác là **lặp nhiều vòng** thay vì một.

Bắt buộc phải có **giới hạn số vòng lặp** (ví dụ `max_steps = 8`) — agent không có phanh sẽ lặp vô hạn, tốn token mà không hội tụ.

## ReAct — Reason + Act

ReAct là khuôn mẫu phổ biến nhất cho agent loop: mỗi bước model sinh ra cặp **Thought** (suy luận) + **Action** (hành động), nhận **Observation** (kết quả), rồi lặp. Ví dụ câu hỏi *"Bộ phim nào của đạo diễn Parasite và năm nào?"*:

```
Thought 1: Mình cần biết đạo diễn phim Parasite là ai.
Action 1:  search("đạo diễn phim Parasite")
Observation 1: Bong Joon-ho
Thought 2: Đạo diễn là Bong Joon-ho. Giờ tìm các phim của ông ấy.
Action 2:  search("phim của Bong Joon-ho")
Observation 2: Memories of Murder (2003), The Host (2006), Parasite (2019)...
Thought 3: Đã có đủ thông tin, trả lời được rồi.
Answer:    Đạo diễn Bong Joon-ho — Memories of Murder (2003), The Host (2006), Parasite (2019)...
```

Điểm mạnh của ReAct: **suy luận và hành động đan xen** — mỗi Observation là dữ liệu thật nên model "đất" lại thực tế, giảm bịa so với suy luận suông. Điểm yếu: mỗi bước tốn một lượt gọi model; với tác vụ đơn giản (một tool call là xong) thì ReAct là thừa.

Với function-calling hiện đại, "Thought" thường được đặt trong trường `reasoning`/`thinking` hoặc ẩn đi, còn "Action" chính là `tool_call` — cấu trúc vẫn y hệt.

Một vòng ReAct tối thiểu trong code — mỗi vòng là một lượt gọi model, tool result nối lại vào `messages`:

```python
for step in range(MAX_STEPS):                    # phanh bắt buộc
    resp = llm.chat(messages, tools=TOOLS_SCHEMA)
    if not resp.tool_calls:                      # không còn Action → có Answer
        return resp.content
    for tc in resp.tool_calls:                   # Action
        result = call_tool(tc.name, tc.arguments)  # Observation
        messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})
raise RuntimeError("Agent không hội tụ trong MAX_STEPS")
```

Điểm dừng của vòng lặp gồm 3 trường hợp: model tự ra câu trả lời cuối (không còn tool call), đạt `MAX_STEPS`, hoặc tool fail lặp lại cùng lỗi 2 lần — cả ba đều phải trả kết quả/hỗi cho người dùng thay vì im lặng.

## Planner vs Executor

Tách hai vai trò là cách nâng agent từ "làm từng bước mù" lên "có kế hoạch":

| | Planner | Executor |
|---|---|---|
| Việc | Phân tích mục tiêu, chia thành danh sách bước | Thực thi từng bước (gọi tool, lấy dữ liệu) |
| Input | Mục tiêu người dùng + trạng thái hiện tại | Một bước trong kế hoạch |
| Output | Danh sách bước (có thể hiển thị cho user duyệt) | Kết quả / lỗi của bước đó |
| Ví dụ | "Cần: 1) tìm giá GPU, 2) so sánh 3 chỗ, 3) gợi ý" | Gọi `get_price()` từng chỗ một |
| Khi fail | **Re-plan** — cập nhật kế hoạch thay vì lì lợm | Trả lỗi lên planner |

Chu trình: **Plan → Execute từng bước → Quan sát → Re-plan nếu cần**. Planner có thể là một prompt riêng ("chia mục tiêu thành các bước"), executor là vòng lặp ReAct con. Lợi ích: kế hoạch nhìn được trước (có thể show cho người dùng duyệt), và khi một bước fail thì chỉ bước đó cần làm lại. Trade-off: tốn thêm token cho planning, và kế hoạch ban đầu có thể sai — vì vậy luôn cho phép re-plan thay vì bám kế hoạch chết.

## LangGraph vs function-calling thuần

**LangGraph** (link ở [file 07](07-resources-and-common-pitfalls.md)) là framework xây agent **có trạng thái**: mô hình hoá flow thành đồ thị gồm nodes (bước xử lý) và edges (điều kiện chuyển bước), trạng thái (state) được truyền qua các node và lưu lại được. Khuyên dùng khi:

- Agent **nhiều nhánh**: cần điều kiện rẽ (nếu thiếu dữ liệu → hỏi lại; nếu đủ → trả lời), lặp có kiểm soát, human-in-the-loop (người duyệt trước khi chạy tool nguy hiểm).
- Cần **bền trạng thái**: dừng/tiếp tục giữa chừng, memory dài hạn, hoặc nhiều agent phối hợp (planner → executor → reviewer).
- Cần **trace rõ ràng**: LangGraph đi kèm LangSmith/Langfuse trace từng node — gỡ lỗi agent phức tạp dễ hơn nhiều.

**Function-calling thuần** (vòng `while` như file 01) đủ khi: agent 1–2 bước, flow thẳng, không cần rẽ nhánh — ít dependency, dễ hiểu, dễ deploy trên VPS 12GB. Nguyên tắc: **đừng đưa framework vào trước khi thật sự cần** — bắt đầu thuần, khi flow phình ra (nhiều nhánh, cần retry có trạng thái, cần duyệt tay) thì chuyển sang LangGraph.

Lưu ý thực tế: cả hai cùng dùng một nền tảng — LangGraph cũng sinh `tool_call` y hệt function-calling thuần; nó chỉ cho bạn **quản lý state + đồ thị** bên trên.

```python
# LangGraph — khung sườn một agent có 2 node và một nhánh điều kiện
from langgraph.graph import StateGraph, END

g = StateGraph(AgentState)
g.add_node("planner", plan_step)      # sinh/cập nhật kế hoạch
g.add_node("executor", act_step)      # gọi tool cho bước hiện tại
g.add_edge("planner", "executor")
g.add_conditional_edges("executor", next_step, {"continue": "planner", "done": END})
app = g.compile()
```

## Liên kết

- [01 — Function/Tool Calling](01-tool-calling-function-calling.md) — nền tảng kỹ thuật của mọi agent loop
- [03 — Memory & MCP](03-memory-and-mcp.md) — cho agent "nhớ" qua các vòng lặp và các phiên
- [05 — Observability & Eval](05-observability-eval-production.md) — đo chất lượng agent bằng eval, không bằng cảm giác
