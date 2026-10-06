---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Agent memory
screen: 21 of 29
screen_type: CHECKPOINT (bài tập tự kiểm tra, khoảng 3 phút)
checkpoint_number: 7
topic: Agent memory
screen_title: Checkpoint 7 - Choose the right memory pattern
related_screens: Screen 19 (Teaching - bảng bốn memory scope), Screen 20 (Watch Out - agent làm đầy window ở session thứ tư)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần "Vì sao các lựa chọn khác sai" là suy luận của Claude, trang chỉ giải thích lý do cho đáp án đúng. Trang đã ở trạng thái hiện đáp án, tôi không bấm Submit
---

# [Module 2 · Screen 21 · CHECKPOINT 7] Chọn đúng memory pattern

## Đề bài và format

Đọc ba use case của agent, ghép **mỗi use case với đúng một memory scope**. Mỗi use case có ba lựa chọn giống nhau:
- **In-context memory:** mọi state sống trong hội thoại đang chạy.
- **External storage:** ghi state vào database cuối session, đọc lại đầu session.
- **No persistent memory (stateless):** mỗi session bắt đầu mới.

Giao diện: ba thẻ use case, bấm chọn một trong ba phương án ở mỗi thẻ, nút **Submit** và **Skip for now**. Sau khi chấm, thẻ hiện "Correct scope" kèm lý do, và khung phản hồi chung ghi "All three use cases matched the right memory scope".

## Đáp án và lý do (theo trang)

| Use case | Scope đúng | Lý do theo trang |
|---|---|---|
| Agent hỗ trợ khách hàng giúp **cùng một người dùng** qua các check-in hằng ngày suốt hai tuần; mỗi session tiếp nối session trước | **External storage** | State in-context mất khi session kết thúc, nên sang ngày hai agent không còn dấu vết check-in hôm qua. Mỗi session sẽ mở đầu như lần liên lạc đầu tiên và người dùng phải giải thích lại tình huống từ đầu |
| Trợ lý code làm việc với developer trong **một session nhiều giờ**, session không tiếp tục sau khi kết thúc | **In-context memory** | External storage là overhead không cần thiết cho session kết thúc khi developer đăng xuất. Một lớp summarized memory sẽ nén mất chi tiết code chính xác mà developer vẫn cần tham chiếu về sau trong cùng session |
| Bộ định dạng tài liệu nhận một file, áp dụng biến đổi, trả kết quả rồi kết thúc; **mỗi job độc lập hoàn toàn** | **No persistent memory (stateless)** | External storage thêm các lời gọi đọc/ghi cho một job không có yêu cầu liên tục. Không gì hỏng, nhưng mỗi lần chạy đều trả chi phí độ trễ và công cài đặt cho state mà agent sẽ không bao giờ dùng lại |

## Vì sao các lựa chọn khác sai (suy luận của Claude, trang không nêu đầy đủ)

**Use case 1 (hỗ trợ khách hàng nhiều ngày)**
- *In-context:* state chỉ sống trong hội thoại đang chạy, nên mất khi session đóng. Có thể nhồi toàn bộ lịch sử hai tuần vào context, nhưng đó là mẫu "mọi thứ trong context" dẫn đến lỗi ở Screen 20 (window đầy, chi phí tăng theo từng lượt).
- *Stateless:* mất toàn bộ ngữ cảnh trước, đúng thứ use case này cần giữ.

**Use case 2 (trợ lý code một session)**
- *External storage:* thêm độ trễ truy xuất và công cài đặt cho state không cần sống sau session.
- *Stateless:* trợ lý sẽ không nhớ code và quyết định của vài giờ trước trong chính session đó, vì "stateless" nghĩa là không có state đáng kể nào được giữ.

**Use case 3 (bộ định dạng tài liệu)**
- *In-context:* chỉ đúng nghĩa trong phạm vi một job; không có giá trị khi mỗi job độc lập và kết thúc ngay, nên không cần thiết kế memory riêng.
- *External storage:* thêm chi phí đọc/ghi cho state không bao giờ được dùng lại.

## Cách nhận biết

Hỏi hai câu: (1) state có cần sống sau khi session kết thúc không? (2) nếu có, session sau có phụ thuộc vào chi tiết cụ thể của session trước không? Không → stateless nếu job độc lập, in-context nếu cần nhớ trong session. Có → external storage.
