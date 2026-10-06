---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Context Engineering
screen: 15 of 29
screen_type: CHECKPOINT (bài tập tự kiểm tra, khoảng 3 phút)
checkpoint_number: 5
topic: Context engineering
screen_title: Checkpoint 5 - Diagnose the context failure
related_screens: Screen 13 (Teaching - pruning, compaction, count_tokens), Screen 14 (Watch Out - cùng mẫu hỏng: tool output tích tụ chen mất chỉ dẫn)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần giải thích vì sao A và C sai là suy luận của Claude dựa trên nội dung bài, vì màn này chỉ hiển thị lý do cho đáp án đúng
---

# [Module 2 · Screen 15 · CHECKPOINT 5] Chẩn đoán lỗi context

## Đề bài, tóm lược

Bài đưa một **session trace** của agent chạy nhiều lượt, trong đó việc chọn tool ngày càng tệ. Người học phải làm ba việc:
1. Chỉ ra **lượt nào gây ra lỗi**.
2. Gọi tên **cơ chế** gây lỗi.
3. Chọn **cách sửa một dòng** trong ba phương án A, B, C.

Mỗi lượt trong trace có phần chi tiết hiện ra khi bấm vào. Đáp án đúng là **B**.

## Session trace và nội dung chi tiết từng lượt

| Lượt | Tool được gọi | Kích thước kết quả | Đánh giá |
|------|---------------|--------------------|----------|
| 1 | `fetch_policy_document` | 2.400 token | Chọn đúng |
| 2 | `fetch_policy_document` | 2.400 token | Chọn đúng |
| 3 | `fetch_policy_document` | 2.400 token | Chọn đúng |
| 4 | `fetch_policy_document` | 2.400 token | Chọn đúng |
| 5 | `search_knowledge_base` thay vì `apply_coverage_rule` | 1.800 token | **Chọn sai** |
| 6 | `search_knowledge_base` lần nữa (lặp lại lượt 5) | 1.800 token | Chọn sai |
| 7 | Phiên kết thúc không có kết quả | không có | Thất bại |

Phần hiện ra khi bấm từng lượt (diễn giải):
- **Lượt 1:** gọi tool đúng, chưa có dấu hiệu gì bất thường.
- **Lượt 2:** vẫn đúng. Output tool tích lũy được 4.800 token.
- **Lượt 3:** vẫn đúng. Output tool tích lũy khoảng 7.200 token.
- **Lượt 4:** **lượt đúng cuối cùng.** Bốn kết quả tool lớn (tổng 9.600 token) đang nằm trong context window và **chen lấn các chỉ dẫn cho Claude biết nên dùng tool nào tiếp theo**.
- **Lượt 5:** **nơi lỗi bắt đầu.** Việc chọn tool không hỏng vì mô tả schema kém: các lượt 1-4 cho thấy phần chọn tool vận hành được. Thứ làm lệch lựa chọn là **ngữ cảnh tích lũy từ bốn lần gọi tool trước đó**.
- **Lượt 6:** lặp lại đúng lỗi của lượt 5. Điều này xác nhận đây là **trôi lệch có hệ thống** do window bị chen chúc, không phải một sự cố ngẫu nhiên.
- **Lượt 7:** phiên không bao giờ hồi phục và kết thúc mà chưa hoàn thành nhiệm vụ.

Kiểm tra số học: 4 × 2.400 = 9.600 token, khớp với con số bài nêu ở Lượt 4.

## Lời giải

- **Lượt kích hoạt lỗi:** lượt **5** (không phải lượt 1).
- **Cơ chế:** **context tích lũy**. Các kết quả tool lớn đã dùng xong vẫn nằm lại trong window, chiếm chỗ và đẩy các chỉ dẫn hiện hành ra sát rìa window, nên model chọn tool kém dần.
- **Cách sửa đúng:** **B.** Cắt tỉa (prune) kết quả `fetch_policy_document` sau mỗi lượt để output tích lũy không chen mất chỉ dẫn hiện hành, và nén (compact) trước lượt 5.

## Các lựa chọn

| | Nội dung | Kết luận |
|---|----------|----------|
| **A** | Thêm mô tả rõ hơn vào schema tool `apply_coverage_rule` | Sai |
| **B** | Cắt tỉa kết quả `fetch_policy_document` sau mỗi lượt để output tích lũy không chen mất chỉ dẫn hiện hành, và nén context trước lượt 5 | **Đúng** |
| **C** | Tăng `max_tokens` trong lời gọi API để Claude có thêm chỗ trả lời | Sai |

Lý do bài đưa ra cho đáp án đúng: lỗi bắt đầu ở lượt 5 chứ không phải lượt 1. Việc chọn đúng tool ở các lượt 1-4 loại trừ nguyên nhân do mô tả schema. Sự chuyển biến ở lượt 5 trỏ về context tích lũy, gồm bốn kết quả tool lớn lấp đầy window và đẩy chỉ dẫn hiện hành ra sát rìa.

## Vì sao A và C sai (phân tích của Claude, không có trong bài)

**Vì sao A sai: sửa nhầm chỗ.**
- A giả định nguyên nhân là mô tả `apply_coverage_rule` chưa đủ rõ, nên model không nhận ra khi nào cần dùng nó.
- Dấu hiệu chống lại giả thuyết này nằm ở **thời điểm lỗi xuất hiện**: bộ tool và system prompt không đổi giữa lượt 4 và lượt 5, nhưng việc chọn tool chỉ hỏng từ lượt 5. Thứ duy nhất thay đổi có hệ thống theo thời gian là lượng context tích lũy.
- Lượt 6 lặp lại đúng lỗi đó cho thấy lỗi gắn với trạng thái context, không phải một lần chọn sai ngẫu nhiên.
- Một mô tả schema tốt hơn có thể cải thiện chút ít, nhưng không đụng tới nguyên nhân gốc: các output cũ vẫn tiếp tục chiếm chỗ và lấn át chỉ dẫn.
- Dạng sai này là bài học lặp lại từ Screen 14: triệu chứng "chọn sai tool" thường bị đọc nhầm, nên kiểm tra context window có đang đầy trước khi debug schema.

**Vì sao C sai: sai loại giới hạn.**
- `max_tokens` giới hạn **độ dài phần model sinh ra** trong một lần trả lời. Nó không giải phóng chỗ trong context cho phần **đầu vào** đang bị tích lũy.
- Triệu chứng ở đây là **chọn sai tool**, không phải câu trả lời bị cắt cụt. Nếu output bị cắt vì `max_tokens`, stop reason sẽ cho thấy điều đó, và đó là hiện tượng khác hẳn.
- Tăng `max_tokens` còn có thể làm tình hình tệ hơn, vì phần dự phòng cho output cũng được tính vào ngân sách tổng của window (mức chắc chắn trung bình: tôi dựa vào cách hiểu thông thường về `max_tokens`, bài không nói rõ điểm này).
- Nó không cắt tỉa, không nén, nên các kết quả tool cũ vẫn chen mất chỉ dẫn.

**Vì sao B đúng:** B tác động thẳng vào cơ chế. Cắt tỉa kết quả tool đã dùng xong để chúng không tiếp tục chiếm chỗ, và nén trước khi chạm mức nguy hiểm. Đây chính là hai chiến lược pruning và compaction ở Screen 13, cũng là cách sửa được dùng trong postmortem ở Screen 14.
