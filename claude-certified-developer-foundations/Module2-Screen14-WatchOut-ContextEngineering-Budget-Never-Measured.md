---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Context Engineering
screen: 14 of 29
screen_type: WATCH OUT (lỗi thường gặp / postmortem, khoảng 5 phút)
topic: Context engineering
screen_title: The session that ran fine in development, then hit a ceiling in production
related_screens: Screen 13 (Teaching - Model selection and keeping multi-turn sessions in budget). Cùng cấu trúc "Watch Out" với Screen 11 (streaming)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Các con số là số liệu của case study trong bài
---

# [Module 2 · Screen 14 · WATCH OUT] Phiên chạy ổn lúc phát triển, chạm trần ở production

## Một câu tóm tắt

Dữ liệu test thường **ngắn hơn** dữ liệu thật, nên ngân sách context mà bạn tưởng dư dả lúc phát triển có thể cạn chỉ sau vài lượt ở production. Triệu chứng lúc đó dễ bị đọc nhầm thành "agent chọn sai tool".

## Bối cảnh (phần "Setup")

Kết quả của tool tiêu tốn context giống hệt prompt và nội dung file đọc vào. Context window là một ngân sách cố định chứa mọi thứ Claude cần thấy ở lượt hiện tại: system prompt, lịch sử hội thoại, và **mọi lần gọi tool cùng kết quả đã tích lũy**.

- Kết quả tool ngắn: mỗi lượt cộng thêm ít, ngân sách dùng được lâu.
- Kết quả tool dài hơn: mỗi lượt cộng thêm nhiều hơn vào cùng một tổng đang chạy, ngân sách cạn nhanh hơn.
- Bản thân cái window không đổi, thứ đổi là **mỗi lượt tiêu tốn bao nhiêu**.
- Hệ quả: phiên xử lý 20 lượt trơn tru ở dev có thể bắt đầu hỏng ở lượt 8 ở production.

> Ví von của Claude: giống một bình xăng cố định dung tích. Chạy thử trên đường phẳng thì tiêu hao ít, bình đủ đi cả chặng. Ra đường đèo thật thì tiêu hao gấp mấy lần, hết xăng giữa đường dù bình vẫn là bình cũ.

## Case study (Postmortem): ngân sách context chưa bao giờ được đo với output tool thật

Diễn biến theo thời gian:

1. **Thiết kế:** một agent xử lý biên lai bán hàng, chạy dưới ngân sách **40k token** cho context. Con số 40k là mức **team tự đặt để kiểm soát chi phí**, không phải giới hạn của model. Bài nhấn mạnh model còn dư địa lớn hơn nhiều: các model Claude API hiện tại có context window tối thiểu 200k token, và các model flagship mới nhất (kể cả Fable) mặc định phục vụ tới 1M token.
2. **Lúc dev:** dùng bộ test gồm 20 biên lai, mỗi lần gọi tool trả về khoảng **800 token**. Cả phiên 20 lượt tốn khoảng **18.000 token**, nằm gọn trong 40k.
3. **Ở production:** biên lai thật đi kèm tài liệu hỗ trợ như bản ghi giao dịch và thư từ. Output trung bình của tool tăng lên khoảng **3.200 token mỗi lần gọi**. Chỉ riêng output tool của 8 lượt đã cộng thành khoảng **25.600 token**. Cộng thêm system prompt, tin nhắn người dùng và tin nhắn assistant thì tổng chạm mức 40k ở **lượt 8**, trước khi agent kịp hoàn thành phân tích.
4. **Triệu chứng đánh lừa:** agent bắt đầu chọn sai tool và trả về phân tích dở dang, trông như lỗi chọn tool.
5. **Nguyên nhân thật:** system prompt và các chỉ dẫn ban đầu bị **chen ra ngoài** bởi các output tool tích tụ mà không bao giờ được dọn sau khi dùng. Agent đang ra quyết định trên một window không còn chứa chính chỉ dẫn mà nó bắt đầu.
6. **Phát hiện:** nhờ một lần kiểm tra mức dùng token (token usage audit), **hai ngày sau khi triển khai**.
7. **Cách sửa:** **cắt tỉa (prune) output tool sau khi đã dùng xong**, và **nén (compact) chủ động trước khi chạm trần**, không đợi đến lúc chạm.

Bảng đối chiếu trong bài:

| | Development | Production |
|---|---|---|
| Context window có sẵn | 200k tiêu chuẩn, 1M trên các model Opus và Sonnet hiện hành (theo bảng) | Như dev |
| Ngân sách team đặt | 40k token | 40k token |
| Output tool trung bình | ~800 token/lần | ~3.200 token/lần |
| Số lượt trước khi đầy | Chạy xong phiên mà không chạm trần | Chạm trần ở lượt 8 |
| Triệu chứng quan sát được | Không có, phiên hoàn tất bình thường | Chọn sai tool và output dở dang từ lượt 8 |
| Nguyên nhân được xác định bằng | Không áp dụng | Kiểm tra mức dùng token, hai ngày sau triển khai |
| Cách sửa | Không áp dụng | Cắt tỉa output tool sau khi dùng và nén chủ động trước khi chạm trần |

## Điều cần nhớ (phần "What to watch out for")

1. **Fixture dev ngắn hơn dữ liệu thật.** Điều này đúng với hầu hết agent được xây dựa trên một bộ fixture. Cách khắc phục: **đo chi phí token thực tế của một kết quả tool với đầu vào lớn nhất mà bạn tìm được trong dữ liệu đích**, trước khi agent ra mắt.
2. **Tràn context thường bị đọc nhầm thành lỗi chọn tool**, vì kết quả nhìn giống nhau. Nếu thấy khả năng chọn tool suy giảm sau một số lượt cố định, hãy **kiểm tra xem context window có đang đầy hay không trước khi mất công debug schema**.

## Phân tích thêm (của Claude, không có trong bài)

- **Liên hệ Screen 11:** hai bài Watch Out cùng một mẫu tư duy: lỗi hiện ra ở chỗ khác với nơi gây ra nó (ở Screen 11 là request kế tiếp, ở đây là việc chọn tool), nên người debug dễ soi nhầm chỗ. Mẹo chung: khi triệu chứng đến muộn và đúng vào một số lượt cố định, nghi ngờ trạng thái tích lũy trước.
- **Liên hệ Screen 13:** phần cách sửa chính là hai chiến lược pruning và compaction ở Screen 13. Công cụ đo trước khi gửi là endpoint `count_tokens` cũng ở Screen 13, rất hợp để kiểm tra giả định ngân sách bằng output tool thật. Việc giữ system prompt và chỉ dẫn quan trọng khỏi bị chen mất là một lý do nữa để viết prompt cho bộ tóm tắt cụ thể.
- **Dấu hiệu nhận biết trong đề thi:** "hỏng sau đúng N lượt" kèm "output tool lớn hơn dữ liệu test" gần như luôn chỉ về áp lực context, không phải schema hay prompt chọn tool.
- **Hai chỗ trong bài mà tôi chưa thấy khớp hoàn toàn (cần đối chiếu, mức chắc chắn trung bình):**
  1. Về số học: ở dev, 20 lượt tốn khoảng 18k trong đó tool chiếm khoảng 16k (20 × 800), tức phần còn lại (system prompt, tin nhắn) khoảng 2k. Nếu phần còn lại ở prod cũng cỡ đó thì 8 lượt chỉ khoảng 25,6k + vài nghìn, chưa tới 40k. Bài nói tổng chạm 40k ở lượt 8 nên có thể phần còn lại ở prod lớn hơn (ví dụ tin nhắn dài hơn) hoặc bài làm tròn để minh họa. Tôi không thể xác nhận từ nội dung bài.
  2. Về window: đoạn văn nói các model flagship mới nhất, kể cả Fable, mặc định phục vụ 1M token, còn bảng ghi "1M trên các model Opus và Sonnet hiện hành". Hai cách nói không giống nhau. Số liệu loại này phụ thuộc phiên bản, nên hãy kiểm tra lại với tài liệu chính thức thay vì dựa vào bài.
