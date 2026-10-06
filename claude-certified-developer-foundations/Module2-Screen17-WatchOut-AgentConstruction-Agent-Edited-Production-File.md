---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Agent construction
screen: 17 of 29
screen_type: WATCH OUT (lỗi thường gặp / postmortem, khoảng 5 phút)
topic: Agent construction
screen_title: The agent that edited a production file
related_screens: Screen 16 (Teaching - bước 4 của checklist nối vòng lặp và bảng điểm chèn HITL), Screen 11 và 14 (các bài Watch Out khác cùng cấu trúc Setup / Case study / What to watch out for)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học
---

# [Module 2 · Screen 17 · WATCH OUT] Agent đã sửa một file production

## Một câu tóm tắt

Vòng lặp chạy đúng như thiết kế, nhưng **điều kiện thoát chỉ kiểm tra file đang sửa**, không có chốt chặn con người trước khi ghi vào môi trường thật. Test trong thư mục tạm không bao giờ lộ ra khoảng trống này.

## Bối cảnh (phần "Setup")

Agent chạy ngon end-to-end khi test vì môi trường test **tha thứ** còn production thì không. Tool, vòng lặp và system prompt giống hệt nhau, chỉ **thiếu checkpoint HITL**, vì lúc test chưa từng xuất hiện tình huống nào cần đến nó.

## Case study

1. **Agent:** đọc, sửa và ghi file cấu hình, có ba tool `read_file`, `write_file`, `validate_config`.
2. **Vòng lặp:** sau mỗi lần ghi, chạy lại `validate_config`; nếu vẫn lỗi thì agent chỉnh và ghi lại, tối đa **mười vòng** rồi dừng.
3. **Lúc test:** chạy trên thư mục tạm với bản sao của config đích. Mọi ca test đều đúng, thường hội tụ về config hợp lệ sau 2-3 vòng.
4. **Ở môi trường khách hàng:** agent nhận ra đúng một tham số vượt phạm vi, đề xuất sửa, gọi `write_file`, chạy lại `validate_config` và nhận pass. Vòng lặp kết thúc sạch chỉ sau **một** vòng, đúng thiết kế. Giới hạn mười vòng không bao giờ bị chạm vì không cần.
5. **Sự cố:** tham số được sửa là một **rate limit mà ứng dụng của khách hàng dựa vào**. `validate_config` chỉ kiểm giá trị mới có nằm trong khoảng cho phép của schema (có). Nó **không kiểm** và không được thiết kế để kiểm xem hệ thống phía dưới có phụ thuộc vào giá trị cũ không. Vài phút sau lần ghi, ứng dụng của khách hàng bắt đầu lỗi vì request bị throttle ở mức nó không được xây để chịu.
6. **Chẩn đoán của bài:** lỗi **không nằm ở vòng lặp**. Điều kiện thoát (`validate_config` trả pass) chỉ có phạm vi **file đang sửa**, và giữa "validation pass trên file này" với "ghi vào môi trường khách hàng" **không có checkpoint nào**.
7. **Phần còn thiếu:** trước lần `write_file` đầu tiên chạm cấu hình sống, vòng lặp phải **tạm dừng và đưa thay đổi đề xuất ra cho con người duyệt**. Nghĩa là cần một nhánh tường minh giữa trạng thái "thay đổi đã sẵn sàng" và "đã ghi", một trạng thái mà developer không thêm vì test không tạo ra ca nào cần nó.

| | Test (thư mục tạm) | Môi trường khách hàng |
|---|---|---|
| Tool, vòng lặp, system prompt | Như nhau | Như nhau |
| Số vòng | 2-3 | 1 |
| Điều kiện thoát | `validate_config` pass | `validate_config` pass |
| Ghi sai thì sao | Không ảnh hưởng gì | Ứng dụng khách hàng lỗi trong vài phút |
| Checkpoint HITL | Không có, không ai thấy thiếu | Không có, và đây là chỗ hỏng |

## Điều cần nhớ (phần "What to watch out for")

- Mẫu này là một **câu hỏi về quyền hạn chưa từng được đặt ra lúc thiết kế**. Agent có quyền ghi vì nhiệm vụ là sửa file, và nhiệm vụ đó hợp lệ. Cái nhóm bỏ sót là **khoảng cách giữa agent đề xuất thay đổi và agent thực thi thay đổi**. Trong môi trường test dùng một lần, khoảng cách này không bao giờ lộ ra vì "ghi sai" không ảnh hưởng gì; production thì khác.
- Câu hỏi thiết kế cần đặt: **"Kết quả xấu nhất là gì nếu `write_file` chạy mà không có người kiểm?"** Câu trả lời quyết định có cần checkpoint HITL trước khi tool chạy hay không.
- Quy tắc: **tool nào có thể thực hiện hành động không hoàn tác được ở production thì cần checkpoint trước khi chạy.** Ghi ràng buộc này vào lúc **thiết kế, khi đang xác định bề mặt tool**, đừng đợi sau sự cố đầu tiên.

## Phân tích thêm (của Claude, không có trong bài)

- **Liên hệ Screen 16:** đây là minh họa trực tiếp cho hàng đầu tiên trong bảng điểm chèn HITL ("trước một tool call phá hủy: thao tác ghi, xóa, gửi; rủi ro cao vì không hoàn tác được") và cho mục 4 trong checklist nối vòng lặp (phải có ít nhất một điểm HITL). Điều kiện thoát (mục 5) thì đã có, nên bài này cho thấy checklist cần **cả hai** mục chứ có điều kiện thoát vẫn chưa đủ.
- **Bài học về phạm vi của việc xác thực:** `validate_config` trả lời đúng câu hỏi của nó (giá trị có hợp lệ theo schema không) nhưng không trả lời câu hỏi mà con người quan tâm (thay đổi này có an toàn với hệ thống đang chạy không). Một bước xác thực tự động chỉ đảm bảo thuộc tính nó được viết ra để kiểm. Dùng "pass" của nó làm tín hiệu "an toàn để commit" là sai phạm vi.
- **Đừng đọc nhầm thành "tăng số vòng hoặc sửa prompt":** vòng lặp thoát sau một lần, đúng ý đồ. Sửa số vòng, prompt hay schema của `validate_config` sẽ trượt trọng tâm; chỗ cần sửa là **cấu trúc của quyền**: tách đề xuất khỏi thực thi. Cách này cũng đứng cùng họ với "nghi nơi khác với nơi triệu chứng xuất hiện" ở Screen 11 và 14.
- **Hướng sửa thực tế (suy luận của Claude, mức chắc chắn trung bình vì bài không nêu chi tiết):** (1) tool ghi ở môi trường thật trả về một "bản đề xuất" (diff) thay vì ghi ngay; (2) chỉ sau khi người duyệt thì mới có lệnh ghi thật; (3) nhìn xa hơn, có thể hạn chế quyền ghi theo môi trường (credentials chỉ-đọc cho production đến khi được duyệt). Bài chỉ yêu cầu "tạm dừng và đưa thay đổi ra cho con người duyệt trước lần ghi đầu tiên".
- **Mẹo nhận dạng trong đề thi:** các dấu hiệu "chạy hoàn hảo khi test", "dùng thư mục/bản sao tạm", "validation pass rồi mà vẫn hỏng ở production", "hành động ghi/xóa/gửi" gần như luôn trỏ về **thiếu HITL ở bước không hoàn tác**, không phải lỗi vòng lặp hay giới hạn số vòng.
- **Độ đầy đủ ghi chú:** tôi đã đọc hết màn này từ đầu đến hết hộp "What to watch out for". Một vài câu giữa các đoạn có thể được tôi diễn đạt lại gọn hơn, nhưng không phát hiện nội dung bị bỏ sót.
