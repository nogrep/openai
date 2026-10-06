---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Agent construction
screen: 18 of 29
screen_type: CHECKPOINT (bài tập tự kiểm tra, khoảng 4 phút)
checkpoint_number: 6
topic: Agent construction
screen_title: Checkpoint 6 - Complete the agent wiring
related_screens: Screen 16 (Teaching - HITL, vòng lặp), Screen 17 (Watch Out - agent sửa file production)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Tôi chỉ đọc đáp án mẫu, không bấm nút tự chấm nào
---

# [Module 2 · Screen 18 · CHECKPOINT 6] Hoàn thiện agent wiring

## Mục tiêu

Kiểm tra bạn nối được hai thứ then chốt của một agent an toàn:
1. **Mô tả tool mang tính hạn chế** để model biết khi nào không được gọi.
2. **Checkpoint HITL trong vòng lặp** để thao tác ghi phải được người duyệt trước khi chạy.

## Format bài

- Trang có: đề bài, một **khối code dở dang** (tools + hàm `run_agent_loop`), hai ô nhập tự do **Gap 1** (viết description) và **Gap 2** (viết code checkpoint).
- Hai nút: **Reveal model answers** (hiện đáp án mẫu) và **Skip for now**.
- Đáp án mẫu hiện trong khung "Model answers · self-assess", kèm lời giải thích, rồi hai nút tự chấm: **Both gaps match · pass** / **Missed one · retry**. Không có chấm tự động, bạn tự so với đáp án.

## Khối code đề cho sẵn (đại ý)

- `tools` gồm `read_record` (đủ mô tả, tham số `customer_id`) và `update_record` (tham số `customer_id`, `field`, `new_value`, **thiếu description = Gap 1**).
- `run_agent_loop`: vòng `while True` gọi API; `end_turn` thì trả kết quả; `tool_use` thì append lượt assistant, duyệt từng block `tool_use`, **[Gap 2: checkpoint trước khi chạy `update_record`]**, chạy `execute_tool`, gom `tool_result`, append một lượt `user`.

## Đáp án mẫu

**Gap 1: description (diễn giải)**: dùng để cập nhật một trường cụ thể của bản ghi khách hàng; chỉ gọi sau khi `read_record` đã xác nhận giá trị hiện tại và thay đổi đề xuất đã được xem xét; không dùng cho cập nhật hàng loạt hay đổi schema.

Bài giải thích: mô tả "hạn chế" nói rõ khi nào không gọi, điều kiện trước khi gọi, và tool không bao giờ dùng cho việc gì. Mô tả trơn thì agent có thể gọi `update_record` ngay cả trước khi đọc giá trị hiện tại hoặc trên trường không định đổi.

**Gap 2: checkpoint HITL**

```python
if block.type == "tool_use":
    if block.name == "update_record":
        print(f"Proposed update, customer_id: {block.input['customer_id']}, "
              f"field: {block.input['field']}, new_value: {block.input['new_value']}")
        approval = input("Approve this update? (yes/no): ").strip().lower()
        if approval != "yes":
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": "Update rejected by operator.",
            })
            continue
    result = execute_tool(block.name, block.input)
```

Bài giải thích: checkpoint nằm **trong vòng lặp** và **gác theo tên tool**, nên `read_record` đi thẳng, `update_record` phải chờ duyệt. Duyệt một lần ở đầu thì không chặn được update cụ thể mà model chưa đề xuất; duyệt sau `execute_tool` thì việc không hoàn tác đã xong.

## Đại ý từng đoạn code đáp án

- **Chọn đúng block và đúng tool:** chỉ `tool_use` tên `update_record` bị chặn, tool đọc không bị.
- **Hiện đề xuất rồi hỏi người:** in bản ghi, trường, giá trị mới; chờ yes/no.
- **Từ chối:** chỉ "yes" mới qua; còn lại thì vẫn trả `tool_result` ("rejected") để không có tool_use mồ côi và để Claude biết mà đổi hướng, rồi `continue` để không chạy tool.
- **Duyệt xong:** mới gọi `execute_tool`.
