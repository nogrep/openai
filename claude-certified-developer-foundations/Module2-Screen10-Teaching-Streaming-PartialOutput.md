---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Streaming responses
screen: 10 of 29
screen_type: TEACHING (bài giảng, khoảng 16 phút)
topic: Streaming responses
screen_title: Streaming responses and handling partial output without corrupting state
related_screens: Screen 11 (Watch Out - tool call viết dở trong history), Screen 12 (Checkpoint 4 - sửa stream handler)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học
---

# [Module 2 · Screen 10 · TEACHING] Streaming responses và xử lý output dang dở

## Một câu tóm tắt

Streaming đổi người lắp ráp câu trả lời từ API sang **code của bạn**, nên bạn phải tự quyết xử lý thế nào khi stream dừng giữa chừng.

## Ý chính: streaming đổi ai là người "lắp ráp" câu trả lời

- **Không streaming:** như đặt hàng một món đồ đã lắp sẵn. API trả về một message hoàn chỉnh, mọi content block đã đầy đủ.
- **Streaming:** như nhận hàng thành nhiều kiện nhỏ gửi lần lượt, kèm hướng dẫn kiện nào ghép vào đâu. Thành phẩm cuối cùng **y hệt** bản không streaming, nhưng **code của bạn là người lắp** và phải tự quyết xử lý ra sao nếu stream dừng giữa chừng.
- Model không "giữ một object sống" cho bạn. Mỗi event chỉ là một mẩu tin mô tả một thay đổi.

## Trình tự event

| # | Event | Ý nghĩa | Handler làm gì |
|---|-------|---------|----------------|
| 1 | `message_start` | Bắt đầu message, content còn rỗng | Chuẩn bị mảng rỗng để chứa các block |
| 2 | `content_block_start` | Mở block mới (text / tool_use / thinking) tại một index | Đặt chỗ cho block. tool_use lúc này có tên và id, **chưa có input** |
| 3 | `content_block_delta` | Một mảnh của block: text, mảnh JSON input, hoặc mảnh thinking | Nối vào đúng block. JSON còn dở, **chưa parse được** |
| 4 | `content_block_stop` | Block hoàn tất | Chốt block. Với tool_use, đây là **lần đầu** JSON đủ để parse |
| 5 | `message_delta` | Thay đổi cấp message: `stop_reason`, usage cuối | Ghi lại `stop_reason` |
| 6 | `message_stop` | Stream kết thúc | Message đã lắp xong, xử lý như response thường |

## Quy tắc cốt lõi: không hành động trên block chưa xong

- Input của tool_use rải qua nhiều delta và **không phải JSON hợp lệ** trước `content_block_stop`.
- Parse hoặc chạy tool sớm dẫn đến lỗi parse, hoặc tool chạy với thiếu tham số.
- Áp dụng cho cả **history**: chỉ thêm lượt assistant vào lịch sử **sau `message_stop`**. Một tool_use dở dang trong history làm request kế tiếp bị từ chối, vì cặp tool_use / tool_result không khớp.

## Khi stream bị đứt giữa chừng

Nguyên nhân thường gặp là mạng rớt, timeout hoặc client ngắt kết nối, khiến stream dừng trước `message_stop`. Hậu quả có hai mức:
- Text dở hiện cho người dùng: **lỗi hiển thị**, khó chịu nhưng vô hại.
- tool_use dở ghi vào history: **lỗi cấu trúc**, làm hỏng hội thoại về sau.

Cách xử lý:
- Stream đứt thì **bỏ hẳn lượt assistant dang dở** và gửi lại request, không lưu nửa vời.
- **Kiểm tra `stop_reason`** trước khi chạy tiếp vòng lặp agent. Chỉ khi giá trị là `tool_use` thì các tool call mới sẵn sàng để chạy. Giá trị khác nghĩa là đang ở nhánh khác.

## Đánh đổi

- **Lợi:** response dài và UI cho người dùng không còn cảnh màn hình trắng chờ đợi.
- **Chi phí:** tự lắp block, không được hành động trên block dở, phải xử lý chủ động trường hợp stream bị ngắt.

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt**

- Streaming đổi người lắp ráp câu trả lời từ API sang **code của bạn**. Thành phẩm cuối giống hệt bản không streaming.
- Trình tự event: `message_start` → `content_block_start` → nhiều `content_block_delta` → `content_block_stop` → `message_delta` (có `stop_reason`) → `message_stop`.
- Input của tool_use đến theo từng mảnh JSON, **chỉ parse được sau `content_block_stop`**. Đừng chạy tool trước đó.
- **Chỉ thêm lượt assistant vào history sau `message_stop`.** Tool_use dở dang trong history làm request kế tiếp bị từ chối.
- Stream đứt giữa chừng: bỏ lượt dang dở, gửi lại request. Kiểm tra `stop_reason == "tool_use"` trước khi chạy tool.
- Text dở chỉ là lỗi hiển thị; tool_use dở là lỗi cấu trúc làm hỏng hội thoại về sau.

**ELI5**

Bạn đặt một cái tủ IKEA, hàng giao thành nhiều thùng nhỏ lần lượt. Bạn là người lắp. Nếu xe giao hàng hỏng giữa đường, thùng cuối không đến, bạn **không** đem cái tủ lắp dở đặt vào phòng khách rồi mời khách ngồi lên. Bạn đợi thùng cuối (`message_stop`), hoặc tháo ra và đặt giao lại.

**Ví dụ khi implement**

**Snippet 1: gom mảnh JSON của tool_use, chỉ parse khi block xong**

```python
if event.type == "content_block_delta" and event.delta.type == "input_json_delta":
    buf[event.index] += event.delta.partial_json      # chỉ nối chuỗi, chưa parse
elif event.type == "content_block_stop":
    tool_input = json.loads(buf[event.index])         # lúc này JSON mới đủ
```

Dùng để: tránh `JSONDecodeError` hoặc chạy tool với thiếu tham số.

**Snippet 2: chốt lượt chỉ sau message_stop**

```python
if event.type == "message_stop":
    messages.append({"role": "assistant", "content": assemble(blocks)})
```

Dùng để: history không bao giờ chứa lượt dở.
