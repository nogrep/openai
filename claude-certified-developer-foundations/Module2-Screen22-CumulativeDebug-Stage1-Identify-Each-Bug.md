---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Cumulative debug task
screen: 22 of 29
screen_type: CUMULATIVE (bài debug tổng hợp, khoảng 8 phút), Stage 1 trong 2 stage
topic: Debug task
screen_title: Cumulative debug task - Identify each bug
related_screens: Screen 10-12 (streaming), Screen 13-15 (context), Screen 16-18 (agent construction), Screen 19-21 (memory). Stage 2 (viết bản sửa) nằm ở màn kế tiếp
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần "Bốn bug" theo model answer của trang
---

# [Module 2 · Screen 22 · CUMULATIVE DEBUG] Stage 1: nhận diện từng bug

## Đề bài

Bản triển khai agent bên dưới có **bốn bug được cài sẵn, mỗi bug ở một trong bốn tầng**: tầng **schema**, tầng **streaming** (nơi response được lắp và commit), tầng **context** (nơi cấu trúc message được dựng) và tầng **memory**. Với mỗi bug: nêu **tên tầng** và viết **một câu** mô tả nó gây ra gì lúc chạy.

Bài chia hai stage: **Stage 1** (màn này) nhận diện bug; **Stage 2** (màn kế) viết phiên bản đã sửa.

## Format bài

Một khối code lỗi (tool definitions, agent loop, memory), một ô nhập tự do cho câu trả lời, nút **Reveal model answer** (chỉ sáng lên sau khi bạn nhập gì đó) và **Skip for now** (hiện thông báo "bạn có thể quay lại checkpoint này bất cứ lúc nào trước cumulative task bằng sidebar").

## Code lỗi (đại ý, viết lại gọn)

```python
# --- TOOL DEFINITIONS ---
tools = [{
    "name": "get_customer_data",
    "description": "Gets data.",
    "input_schema": {"type": "object",
                     "properties": {"id": {"type": "string"}},
                     "required": ["id"]},
}]

# --- AGENT LOOP ---
def run_agent(user_request, session_history):
    messages = session_history + [{"role": "user", "content": user_request}]
    while True:
        blocks = {}
        stop_seen = False
        with client.messages.stream(model=model, max_tokens=4096, tools=tools,
                                    messages=messages,
                                    thinking={"type": "adaptive"}) as stream:
            for event in stream:
                if event.type == "content_block_start":
                    blocks[event.index] = init_block(event)
                elif event.type == "content_block_delta":
                    apply_delta(blocks[event.index], event.delta)
                elif event.type == "message_stop":
                    stop_seen = True
        assistant_content = [b for b in assemble(blocks) if b["type"] != "thinking"]
        messages.append({"role": "assistant", "content": assistant_content})
        response = finalize(blocks)
        if response.stop_reason == "end_turn":
            return response
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                messages.append({"role": "user", "content": [
                    {"type": "tool_result", "tool_use_id": block.id, "content": result}]})

# --- MEMORY ---
def build_session_history(prior_sessions):
    # Concatenating all prior session transcripts in-context
    full_history = []
    for session in prior_sessions:
        full_history.extend(session["messages"])
    return full_history
```

## Bốn bug (theo model answer của trang)

| Tầng | Bug trong code | Hậu quả lúc chạy |
|---|---|---|
| **Schema** | Mô tả tool chỉ là `"Gets data."` | Claude không phân biệt được tool này với các tool truy xuất khác nên chọn tool theo mức khớp bề mặt chứ không theo ý định |
| **Streaming** | Lượt assistant được commit vào history **trước khi** thấy `message_stop`, và khối `thinking` bị loại bỏ | Stream bị ngắt sẽ ghi một `tool_use` dở dang vào history; ngoài ra bỏ khối thinking vi phạm quy tắc carry-back, nên API từ chối request kế tiếp vì chữ ký (signature) không còn khớp |
| **Context** | Chỉ append `tool_result`, không có lượt assistant chứa `tool_use` đứng trước nó | API thấy một `tool_result` tham chiếu tới `tool_use` mà nó chưa từng nhận như một lượt assistant hoàn chỉnh, nên từ chối request |
| **Memory** | Nối toàn bộ transcript các session trước vào context | Context window lớn dần theo từng session và đầy trước khi agent kịp xử lý yêu cầu hiện tại, vào khoảng session bốn hoặc năm |

Trang kết thúc bằng hai nút tự chấm: "I found all four · pass" và "I missed one or more · retry".

## Đối chiếu với bản phân tích trước của tôi (Claude)

- **Schema và Memory:** khớp đáp án.
- **Streaming:** tôi chỉ bắt được nửa đầu (commit trước `message_stop`, `stop_seen` không được dùng). Tôi từng ngờ việc loại khối `thinking` là vấn đề phụ, nhưng đáp án coi nó là một phần của bug streaming: khối thinking phải được giữ nguyên khi trả lại lượt có `tool_use`.
- **Context:** tôi đoán sai hướng (mỗi `tool_result` một tin nhắn `user` riêng). Đáp án nói về việc thiếu lượt assistant chứa `tool_use` trước `tool_result`. Đoạn code ở trên là bản tôi viết gọn lại từ trang, nên chi tiết phần dựng message có thể không giống code gốc; hãy tin đáp án của trang.
