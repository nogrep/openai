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
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Trang KHÔNG hiện đáp án mẫu nếu bạn chưa nhập câu trả lời (nút Reveal model answer bị khóa), nên phần "Bốn bug" bên dưới là phân tích của Claude từ code, không phải đáp án chính thức. Tôi đã bấm "Skip for now" một lần để thử, nút này chỉ bỏ qua và không hiện đáp án
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

## Bốn bug (phân tích của Claude, không phải đáp án chính thức)

| Tầng | Bug | Hậu quả lúc chạy |
|---|---|---|
| **Schema** | Mô tả tool chỉ là `"Gets data."`: không nói tool lấy dữ liệu gì, khi nào gọi, `id` là loại id nào, khi nào không dùng | Model không có căn cứ định tuyến; gọi sai lúc, sai tool hoặc truyền sai `id` (liên hệ Screen 16 và Checkpoint 6 về mô tả tool hạn chế) |
| **Streaming** | Biến `stop_seen` được set khi thấy `message_stop` nhưng **không bao giờ được đọc**; lượt assistant được append vào history vô điều kiện | Nếu stream đứt giữa chừng, một lượt dở (có thể kèm tool_use nửa vời) bị ghi vào history và request kế tiếp bị từ chối (liên hệ Screen 10-12) |
| **Context** | Mỗi `tool_use` được trả bằng **một tin nhắn `user` riêng** chứa một `tool_result` | Khi Claude gọi nhiều tool trong cùng một lượt, các `tool_result` phải được trả **cùng nhau trong một lượt `user`**; tách riêng làm cấu trúc message sai và request bị từ chối hoặc hành vi lệch (liên hệ checklist nối vòng lặp ở Screen 16) |
| **Memory** | `build_session_history` **nối toàn bộ transcript các session trước vào context** | Context phình theo số session, tốn token và chạm trần window sớm như ở Screen 20; cần external storage và chỉ chèn tập con liên quan hoặc bản tóm tắt |

Mức chắc chắn: bốn bug trên khớp rõ với bốn tầng và với các màn trước nên tôi khá chắc. Hai điều tôi **không** chắc:
- Dòng `if b["type"] != "thinking"` loại các khối thinking khi lưu lượt assistant có thể là một vấn đề phụ (với extended thinking, khối thinking thường cần được giữ nguyên khi trả lại lượt có tool_use), nhưng tôi không biết bài có coi đây là bug chính thức hay không. Hãy đối chiếu tài liệu extended thinking.
- Cách bài diễn đạt từng bug (ví dụ có nhấn thêm vào `finalize(blocks)` hay `end_turn`) có thể khác cách tôi viết.

## Cách tận dụng màn này

Gõ câu trả lời của chính bạn vào ô, rồi bấm **Reveal model answer** để so với đáp án chính thức; sau đó đối chiếu với bảng trên và cho tôi biết nếu đáp án mẫu khác, tôi sẽ cập nhật file.
