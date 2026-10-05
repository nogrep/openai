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

## Phân tích thêm (của Claude, không có trong bài)

- **Liên hệ các screen sau:** quy tắc "chỉ thêm vào history sau `message_stop`" ở đây chính là điều mà case study ở Screen 11 vi phạm, và là lỗi cần sửa trong Checkpoint 4 ở Screen 12.
- **Độ đầy đủ của ghi chú:** khi đọc màn này tôi cuộn nhanh nên có một đoạn ngắn nằm giữa phần "quy tắc cốt lõi" và phần "khi stream bị đứt" chưa được đọc kỹ. Các ý chính ở hai phần đó vẫn đủ, nhưng nếu bạn thấy thiếu ý thì đối chiếu lại với màn gốc.
