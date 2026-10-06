---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
screen: 12 of 29
screen_type: CHECKPOINT (bài tập tự kiểm tra, thời lượng khoảng 4 phút)
checkpoint_number: 4
topic: Streaming responses
screen_title: Checkpoint 4 - Repair the broken stream handler
related_screens: Screen 10 (Teaching - streaming và partial output), Screen 11 (Watch Out - tool call viết dở trong history)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Đoạn code bên dưới do Claude viết lại, không chép từ bài.
---

# [Module 2 · Screen 12 · CHECKPOINT 4] Sửa stream handler bị lỗi

## Đề bài, tóm lược

Bài đưa ra một handler (viết bằng Python) làm hai việc: nhận response dạng stream, rồi thêm lượt assistant vào lịch sử hội thoại. Handler này có **đúng một lỗi** và lỗi chỉ lộ ra khi stream bị ngắt giữa chừng. Nhiệm vụ: chỉ ra lỗi và viết lại bản đúng. Màn này có ô nhập code, và sau khi bạn làm xong thì đối chiếu với đáp án tham chiếu rồi tự chấm "khớp" hoặc "chưa khớp".

## Handler lỗi hoạt động thế nào (mô tả bằng lời)

1. Khởi tạo một dict rỗng để chứa các block, và một cờ boolean đặt là "chưa thấy stop".
2. Mở stream rồi duyệt từng event:
   - gặp `content_block_start`: tạo block tại index tương ứng.
   - gặp `content_block_delta`: nối mảnh delta vào block ở index đó.
   - gặp `message_stop`: bật cờ lên "đã thấy stop".
3. Sau khi vòng duyệt kết thúc, **luôn luôn** lắp các block thành nội dung và thêm lượt assistant vào `messages`.

## Lỗi nằm ở đâu

Cờ đã được bật đúng chỗ, nhưng **không ai dùng nó**. Bước thêm vào history chạy vô điều kiện, bất kể có nhận được `message_stop` hay không.

Hệ quả: nếu stream bị ngắt, handler vẫn commit một lượt dang dở, có thể chứa một block tool_use mới có nửa JSON input. Đây chính xác là kịch bản đã gây sự cố trong case study ở Screen 11.

Một chi tiết đáng chú ý: lỗi này **vô hình khi test trên kết nối tốt**, vì lúc nào `message_stop` cũng đến và việc thêm vô điều kiện vẫn cho kết quả đúng. Nó chỉ nổ ở production khi có gián đoạn.

## Cách sửa

Biến cờ thành điều kiện chặn:
- **Có `message_stop`:** lắp block và thêm vào history như bình thường.
- **Không có `message_stop`:** không thêm gì vào history, báo lỗi (raise) để tầng gọi **retry từ lượt hoàn chỉnh gần nhất**.

Phiên bản viết lại của Claude (cùng ý, khác cách viết so với bài):

```python
blocks = {}
finished = False

with client.messages.stream(
    model=model, max_tokens=4096, messages=messages, tools=tools
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            blocks[event.index] = init_block(event)
        elif event.type == "content_block_delta":
            apply_delta(blocks[event.index], event.delta)
        elif event.type == "message_stop":
            finished = True

if not finished:
    # Không ghi gì vào history. Để lớp ngoài retry từ lượt hoàn chỉnh gần nhất.
    raise StreamInterruptedError("Stream ended before message_stop; partial turn discarded")

messages.append({"role": "assistant", "content": assemble(blocks)})
```

## Nguyên lý rút ra

- Việc "vòng đọc đã thoát" và việc "message đã hoàn tất" là **hai sự kiện khác nhau**. Điều kiện commit phải dựa vào tín hiệu hoàn tất của giao thức (`message_stop`).
- Một biến trạng thái được set mà không bao giờ được đọc là dấu hiệu code smell. Khi review, nếu thấy cờ kiểu `xxx_seen` mà không có nhánh nào rẽ theo nó, hãy nghi ngờ ngay.
- Thà **bỏ nguyên lượt dang dở và retry** còn hơn lưu nửa vời, vì lỗi do lưu nửa vời sẽ hiện ở request sau và rất khó truy ngược.
