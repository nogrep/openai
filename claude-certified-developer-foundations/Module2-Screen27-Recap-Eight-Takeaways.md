---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Module wrap up
screen: 27 of 29
screen_type: RECAP (tổng kết module)
topic: Recap - eight takeaways, one per enabling objective
screen_title: "Recap: Eight takeaways, one per enabling objective"
related_screens: Toàn bộ Module 2 (Screen 1 đến 25)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần đầu bám đủ nội dung màn gốc, phần "Tóm tắt, ELI5 và ví dụ" ở cuối là phần Claude thêm
---

# [Module 2 · Screen 27 · RECAP] Tám điều cần nhớ, mỗi điều ứng với một mục tiêu của module

## 1. Prompt hỏng: kiểu hỏng cho biết đang thiếu kỹ thuật nào

| Triệu chứng | Thiếu gì |
|---|---|
| Output sai hình dạng | Thiếu ràng buộc output |
| Trôi dần qua nhiều lượt | System prompt chưa đủ cụ thể |
| Bịa ra cấu trúc | Thiếu ví dụ few-shot |

Viết lại câu lệnh rồi thử lại hiếm khi được, vì đây không phải lỗi câu chữ. Hãy **chẩn đoán kiểu lỗi trước**, rồi thêm đúng kỹ thuật cho kiểu lỗi đó.

Nếu dặn trong prompt vẫn không đủ (ví dụ có input chưa test làm hỏng parser), hãy chuyển việc kiểm soát output vào API bằng **structured outputs**:

- **JSON outputs** ràng buộc câu trả lời cuối theo schema.
- **Strict tool use** kiểm tra tham số Claude truyền vào tool.

Cái giá: **latency biên dịch ở lần gọi đầu** và **tốn thêm input token**.

## 2. Chỉnh độ sâu suy luận theo bài toán trước khi tinh chỉnh prompt

- Chỉ bật reasoning ở chỗ một lượt suy luận **thật sự làm đổi câu trả lời**.
- Chỉnh **effort** theo bài toán, không tăng cho mọi call.
- **Thinking blocks phải được gửi lại API nguyên vẹn**, nếu không request kế tiếp sẽ lỗi.
- Việc chọn model nào (khác với việc có bật reasoning hay không) được dạy ở module MSO Foundations, đứng trước module này.

## 3. Stream kết thúc không có nghĩa là message đã hoàn tất

Streaming giúp người dùng thấy phản hồi nhanh hơn, đổi lại bạn phải tự ráp response từ các event rời.

- Chỉ xử lý một block khi block đó **đã đóng**.
- Chỉ ghi lượt vào history **sau `message_stop`**.
- Stream đứt giữa chừng: **bỏ lượt dang dở rồi retry**.

Kiểu lỗi cần nhận ra: retry bị lỗi tool use, nhưng nguyên nhân là một block ráp dở từ stream bị rớt, **không phải do schema**.

## 4. Mọi lần chọn sai tool đều truy về schema, và phần lớn là do description

Claude chọn tool bằng cách đọc description rồi so với yêu cầu. Hai tool cùng ghi "dùng để tìm thông tin" thì với Claude là không phân biệt được, dù input schema khác hẳn nhau.

Câu sửa được phần lớn bug chọn sai tool là **điều kiện loại trừ**: mỗi description có một dòng "khi nào KHÔNG gọi tool này", viết **ngay lúc thiết kế**, đừng đợi thấy call sai trong log.

Nếu tool đã có người viết sẵn, có thể kết nối một **MCP server đang được bảo trì** thay vì tự viết schema. Nhưng mỗi server đều đưa tool definition vào context **dù tool có được dùng hay không**, nên chỉ kết nối khi cần và kiểm soát chi phí nạp.

## 5. Context là ngân sách cố định, và tool output tiêu nó nhanh nhất

- Ở production, tool output dài gấp **3 đến 5 lần** dữ liệu mẫu lúc dev.
- Session chạy gọn 50 lượt khi test có thể **đầy context ở lượt 8** khi đã ship.
- **Pruning**, **compaction** và **subagent handoff** đều lấy lại chỗ, mỗi cách một kiểu. Chọn cách nào tùy bạn còn cần state cũ hay không.
- Khi Claude chọn tool kém dần sau một số lượt cố định, hãy xem **context window trước**, không phải schema.

## 6. Workflow hay agent quyết định chi phí của mọi thứ phía sau, và checkpoint của con người là một phần của thiết kế

- **Workflow:** khi bạn viết được chính xác từng bước bằng code.
- **Agent:** khi bạn biết mục tiêu và có tool, nhưng không biết trước đường đi.

Chọn sai theo hướng nào cũng chỉ lộ ra ở production:

- Dùng agent khi workflow là đủ thì **tốn context**, và hành vi nằm trong transcript.
- Dùng workflow khi cần agent thì **gãy ngay lần đầu** gặp input nằm ngoài đường đã viết.

Tool nào làm được việc **không hoàn tác** thì checkpoint human-in-the-loop (HITL) phải có **trước khi nối vòng lặp**, không phải sau khi lần ghi đầu tiên đã chạm vào môi trường của khách.

## 7. Scope của memory do hình dạng session quyết định, không phải do cái gì dễ code nhất

| Kiểu | Đặc điểm |
|---|---|
| In-context | Dễ viết nhất nên cũng hỏng sớm nhất, khi session ở production hóa ra ngắn hơn và nhiều hơn so với lúc dev |
| External storage | Thêm latency nhưng state sống qua nhiều session |
| Summarized | Rẻ hơn nhưng mất những gì prompt tóm tắt không giữ |
| Stateless | Đúng cho job chạy xong là đóng |

- Sửa gấp từ in-context sang external khi đã chạy thật mất khoảng **1 giờ**; chọn đúng lúc thiết kế mất khoảng **20 phút**.
- Mang **hướng dẫn lặp lại** qua nhiều task là chuyện khác với mang **state**. Pattern cho việc đó là **Skill**: một file markdown Claude nạp khi cần bằng cách khớp description, thay vì nhồi vào mọi session.

## 8. Tính chi phí input đa phương thức trước khi viết code ingestion, và chọn API theo khối lượng việc

Một ảnh tốn **⌈rộng/28⌉ × ⌈cao/28⌉ visual token**, và mức trần mỗi ảnh khác nhau theo tầng model. Ảnh gốc độ phân giải cao trên model mới có thể tốn gấp nhiều lần thumbnail trong bộ test, nên hãy tính với **input lớn nhất dự kiến gặp ở production**.

| Tình huống | Chọn |
|---|---|
| Ảnh dùng một lần | Base64 inline |
| Asset dùng lại nhiều lần | Files API |
| Việc offline | Message Batches API: rẻ hơn mỗi token nhưng không biết chính xác khi nào xong |

Lỗi cần tránh: gọi API đồng bộ trong vòng lặp rồi coi đó là batching.

## What comes next

Module này dựng thư viện primitive cho Developer, gồm **5 kiểu tương tác** mà mọi module Developer sau đều dùng. Nền tảng gồm: prompting craft, tool schema, context engineering, xây agent, scope memory và ingestion đa phương thức.

Kết quả của module: bạn đưa được prototype dùng Claude lên production, với prompt, vòng lặp tool use, streaming, context/memory và vòng lặp agent có checkpoint, **tất cả chịu được tải thật**.

## Sources (trên trang)

- Claude 101
- Claude Code 101 In Action (`/compact`, `/clear`, vòng lặp agent)
- AI Fluency Framework Foundations
- Building with the Claude API
- platform.claude.com (trang ghi: lấy lại và kiểm tra lại khi xuất bản)
- Blog "Building Effective Agents" (chaining, routing, parallelization, evaluator-optimizer)

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt một câu:** làm app Claude chịu được tải thật nghĩa là chẩn đoán đúng kiểu lỗi, tiết kiệm context, đặt tên và mô tả tool kỹ, ráp stream cẩn thận, có người duyệt trước việc không hoàn tác, chọn đúng kiểu nhớ và tính tiền ảnh từ trước.

**ELI5 theo từng ý:**

1. **Chẩn đoán trước khi kê thuốc:** giống đi khám, bác sĩ hỏi triệu chứng rồi mới kê đơn, không phải uống đại thuốc khác nhãn. Cần form cố định thì dùng structured outputs, như phát phiếu có sẵn ô điền.
2. **Reasoning là giấy nháp:** bài dễ không cần nháp, bài khó mới nháp. Giấy nháp phải nộp lại nguyên vẹn (thinking blocks) cho lượt sau.
3. **Streaming như nghe điện thoại:** bị ngắt giữa câu thì chưa phải người ta nói xong. Chỉ ghi vào sổ khi nghe "hết rồi" (`message_stop`).
4. **Description là biển chỉ dẫn:** hai biển cùng ghi "Lối vào" thì không ai biết đi cửa nào. Mỗi biển cần thêm "không đi cửa này nếu…". MCP server như thuê sẵn nhân viên của hãng khác: tiện nhưng mỗi người ngồi sẵn trong phòng họp (context) dù chưa được gọi.
5. **Context là cái ví:** lúc test mỗi lần chi 1 đồng, lúc chạy thật chi 3 đến 5 đồng nên hết ví ở lượt 8. Lấy lại chỗ bằng cách bỏ bớt (pruning), tóm tắt (compaction) hoặc nhờ người khác làm hộ (subagent).
6. **Workflow là công thức, agent là đầu bếp tự xoay xở.** Việc không hoàn tác (xóa, chuyển tiền) thì cần người duyệt trước, như bếp trưởng nếm trước khi bưng ra bàn.
7. **Memory:** in-context như nhớ trong đầu, external như ghi sổ, summarized như ghi tóm tắt, stateless như làm xong là quên luôn. Skill như cuốn sổ tay hướng dẫn mà bạn chỉ mở đúng trang khi cần.
8. **Ảnh tính tiền theo kích thước:** tính với ảnh to nhất sẽ gặp, không phải ảnh test.

**Ví dụ tính token ảnh (Claude tự tính, trang không in số này):**

- Ảnh test 280 × 280: ⌈280/28⌉ × ⌈280/28⌉ = 10 × 10 = **100** visual token.
- Ảnh thật 1568 × 1568: ⌈1568/28⌉ × ⌈1568/28⌉ = 56 × 56 = **3136** visual token, gấp hơn 31 lần ảnh test.
- Mức trần mỗi ảnh còn tùy tầng model, nên chỉ coi đây là minh họa công thức.

**Mẹo ôn thi:** khi đọc một lỗi trong đề, tự hỏi nó thuộc ý nào trong 8 ý trên. Sai hình dạng output là ý 1, retry hỏng tool use sau stream là ý 3, chọn tool kém dần theo số lượt là ý 5, và job "batch" gọi đồng bộ trong vòng lặp là ý 8.
