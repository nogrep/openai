---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Context Engineering (phần mới, sau phần Streaming responses)
screen: 13 of 29
screen_type: TEACHING (bài giảng, khoảng 16 phút)
topic: Context engineering
screen_title: Model selection and keeping multi-turn sessions in budget
related_screens: Screen 14 (dự kiến là postmortem về agent chạy tốt trên test rồi chạm trần context ở production, theo "Forward pointer" cuối Screen 13)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Các ví dụ ví von (analogy) là của Claude. Con số và tên model là theo bài tại thời điểm đọc, bài tự dặn phải kiểm tra lại với tài liệu chính thức khi build.
---

# [Module 2 · Screen 13 · TEACHING] Chọn model và giữ phiên nhiều lượt trong ngân sách context

## Bức tranh lớn

Có hai ràng buộc nối tiếp nhau:
1. **Chọn model** là quyết định sớm nhất, và nó đặt "sàn" về giá và tốc độ cho mọi quyết định sau đó.
2. **Context window** là ràng buộc kế tiếp: toàn bộ chữ mà model đọc được trong một lần, gồm prompt, lịch sử hội thoại và mọi kết quả tool.

Mọi kết quả tool mà Claude nhận về đều được thêm vào window và **ở lại đó đến hết phiên**. Với một prompt một lượt thì điều này vô hình. Với agent chạy 10-20 lần gọi tool thì window đầy rất nhanh, và khi đầy thì agent hoặc phải nén (mất chi tiết) hoặc đứng khựng giữa chừng.

**Context engineering** = quyết định trước: cái gì được vào window, cái gì quay ra dưới dạng bản tóm tắt, và cái gì không bao giờ được vào.

> Ví von của Claude: window giống mặt bàn làm việc. Mỗi tài liệu bạn đặt lên là chiếm chỗ cho đến hết buổi. Context engineering là việc quyết định trước tài liệu nào được đặt lên bàn, tài liệu nào chỉ giữ lại bản ghi chú, và tài liệu nào để nguyên trong tủ.

## 1. Chọn model: bắt đầu từ Sonnet, đổi có chủ đích

Theo bài, họ Claude hiện có bốn tầng: **Fable, Opus, Sonnet, Haiku**.

| Tầng | Vai trò theo bài |
|------|-------------------|
| Sonnet | Mặc định cân bằng cho phần lớn workload production |
| Haiku | Nhanh và rẻ, cho tác vụ nằm trong khả năng của nó |
| Opus | Việc khó hơn mức Sonnet đáp ứng |
| Fable | Mạnh nhất của Anthropic, cho tác vụ đòi hỏi cao nhất như suy luận phức tạp, coding nâng cao, tổng hợp nghiên cứu, workflow agent tinh vi |

Quy tắc:
- Điểm xuất phát là **Sonnet**.
- **Lên Opus** chỉ khi bộ eval cho thấy Sonnet không đạt chuẩn chất lượng của bạn.
- **Xuống Haiku** chỉ khi eval cho thấy mức giảm chất lượng là chấp nhận được **với tác vụ của bạn**, không chỉ vì muốn tiết kiệm.
- Quyết định đổi model phải là quyết định **đo lường được**.
- Bài dặn kiểm tra danh sách model và mã định danh hiện hành trên platform.claude.com/docs lúc build.

## 2. Context window không phải tài nguyên miễn phí

- Mọi thứ chiếm chỗ: mỗi tin nhắn, kết quả tool, tài liệu chèn vào và mỗi câu trả lời của Claude.
- **Hai kiểu hỏng khi chạm trần** (và cả hai đều không âm thầm cắt nội dung cũ):
  - Request **đã lớn hơn window ngay từ đầu**: Messages API từ chối bằng lỗi validation, trước khi sinh nội dung.
  - Request vừa nhưng **sinh ra đến nửa chừng thì chạm trần**: các model hiện tại trả về phần đã sinh kèm stop reason `model_context_window_exceeded`.
- Muốn phiên chạy tiếp qua giới hạn thì **ứng dụng của bạn phải tự cắt hoặc tóm tắt history** trước request kế tiếp.
- **Dev khác prod:** khi phát triển, input nhỏ và phiên ngắn nên window hiếm khi đầy. Ở production, output của tool thường dài gấp 3-5 lần dữ liệu test, phiên chạy nhiều lượt hơn, và window đầy ở khoảng lượt 8 thay vì lượt 50. Không lập kế hoạch cho việc này thì cái giá là một sự cố production.

## 3. Bốn chiến lược giữ trong ngân sách

Lý do nền tảng: mỗi token trong window tốn tiền input và cộng thêm độ trễ, và phiên dài làm cả hai cộng dồn.

| Chiến lược | Làm gì | Dùng khi | Mất gì |
|-----------|--------|----------|--------|
| **Pruning** (cắt tỉa) | Quay lại một tin nhắn trước đó và tiếp tục từ đó, bỏ đoạn hội thoại sau nó | Claude đã đi vào ngõ cụt, hoặc đã tích tụ nhiều vòng debug qua lại không còn giúp ích cho việc kế tiếp | Mọi công việc sau điểm quay lại. Nếu Claude học được điều hữu ích trong đoạn đó thì phải học lại |
| **Compaction** (nén) | Tóm tắt lịch sử thành bản gọn giữ lại thông tin chính. Trong Claude Code là `/compact`. Ở API là server-side compaction (beta, nền tảng làm giúp), hoặc tự tóm tắt phía client | Phiên sắp chạm trần nhưng bạn muốn làm tiếp cùng tính năng với kiến thức Claude đã tích lũy | Chi tiết có thể rơi mất. Cái gì không có trong bản tóm tắt thì Claude sẽ không còn biết |
| **Clearing** (xóa sạch) | Bắt đầu hội thoại mới với context rỗng (`/clear` trong Claude Code, session mới ở API) | Việc kế tiếp hoàn toàn khác, và context cũ chỉ gây thiên lệch hoặc nhiễu | Toàn bộ context phiên. Thứ gì cần nhớ xuyên phiên phải đặt ở nơi bền vững như file `CLAUDE.md` |
| **Subagent handoffs** (giao việc cho subagent) | Tạo subagent trong window cô lập, chỉ có mô tả tác vụ và system prompt cần thiết. Subagent làm xong và trả về bản tóm tắt | Subtask đủ tự chứa, đặc biệt là việc khám phá mà quá trình làm rối context chính còn đáp án thì ngắn | Khả năng nhìn thấy cách subagent đi đến kết luận. Các bước trung gian bị bỏ cùng context của nó |

## 4. Hai đòn bẩy khác: prompt caching và token counting

Bốn chiến lược trên quản lý **cái gì vào window**. Hai tính năng API dưới đây giảm **chi phí cho phần đã có sẵn** trong window.

**Prompt caching**
- Lưu phần xử lý của một **tiền tố ổn định** trong request, để các request sau dùng lại thay vì xử lý lại cùng số token.
- Request đầu ghi tiền tố vào cache. Các request sau gửi **đúng nội dung giống hệt** đến điểm đó thì trả một phần nhỏ chi phí gốc.
- Ứng viên mạnh nhất: system prompt dài, bộ định nghĩa tool lớn, tài liệu tham chiếu hỏi đi hỏi lại, tức những phần ít đổi giữa các lượt.
- Cách bật: đánh dấu điểm ngắt cache bằng trường `cache_control` kiểu `ephemeral` trên **block cuối cùng** bạn muốn cache. Được đặt **tối đa bốn** điểm ngắt.
- Với phiên nhiều lượt có system prompt và schema tool ổn định, cache các tiền tố đó một lần rồi dùng lại là cách giảm chi phí hiệu quả nhất theo bài.

**Token counting**
- Đo áp lực context **trước khi** gửi request, thay vì sau khi lỗi.
- Endpoint `count_tokens` nhận cùng thân request như một lời gọi messages và trả về số token mà **không chạy suy luận**.
- Dùng khi phát triển để kiểm chứng giả định ngân sách với output tool thật (không chỉ dữ liệu test), và ở production để chặn các request sắp vượt window trước khi chúng lỗi.

## 5. Ba chỗ đường RAG có thể hỏng

| Khâu | Nó quyết định gì | Cạm bẫy |
|------|------------------|---------|
| **Chunking** (chia đoạn) | Một đơn vị ngữ cảnh truy xuất là gì | Chia quá nhỏ: một chunk thiếu ngữ cảnh xung quanh nên vô dụng. Chia quá lớn: chunk loãng, kéo theo chữ không liên quan. Mặc định hợp lý là chia theo câu hoặc theo mục kèm một chút chồng lấn, vì sự kiện nằm vắt qua ranh giới sẽ bị tách đôi nếu không chồng lấn |
| **Embedding match** | Chunk nào được trả về | Dùng tìm kiếm tương tự nên lấy nội dung gần nghĩa, không nhất thiết chứa đúng thuật ngữ bạn cần. Truy vấn một mã định danh cụ thể có thể trượt chunk đúng nếu có kết quả gần nghĩa hơn xếp trên. Vì vậy đôi khi chạy thêm khớp từ vựng song song |
| **Assembly** (lắp vào prompt) | Chunk truy xuất có đến được model theo cấu trúc prompt mong đợi hay không | Sai cấu trúc thì model trả lời từ trí nhớ thay vì từ văn bản vừa truy xuất |

So sánh hai hướng truy xuất:
- **Fetch-once (có index):** hệ thống dễ suy luận, kiểm tra được chunk nào được lấy cho một truy vấn và test truy xuất trực tiếp. Cái giá là hạ tầng: xây index, lưu, giữ đồng bộ khi dữ liệu đổi, và bảo mật nơi lưu nó.
- **Search-across-rounds (tìm kiếm lặp, agentic search):** bỏ hạ tầng và nguy cơ dữ liệu cũ, vì model đọc file hiện hành lúc truy vấn. Cái giá là tốn thêm token và thời gian mỗi truy vấn, và quy trình khó quan sát hơn.
- Kho tham chiếu ổn định với truy vấn tra cứu đơn giản: đáng giữ index. Kho thay đổi hoặc câu hỏi nhiều bước: tìm kiếm lặp thường là hệ thống đơn giản hơn dù đắt hơn mỗi truy vấn.
- Con số "mức tăng hiệu năng của agentic search đơn agent so với retrieval index" trong bài là con số **gắn với một phiên bản**. Bài dặn xác nhận lại ở tầng tài liệu tham chiếu lúc build thay vì tin con số trong module.

## 6. Compaction: giữ được gì phụ thuộc cách bạn viết summarizer

- Với `/compact` trong Claude Code, công cụ tự quyết nội dung bản tóm tắt.
- Ở API, chiến lược chính được tài liệu hóa là **server-side compaction (beta)**: nền tảng tóm tắt hội thoại khi bạn cấu hình trong request.
- Nếu **tự nén thủ công** thì bạn tự viết prompt cho bộ tóm tắt, và prompt đó quyết định agent biết gì ở các lượt sau.

Bài đặt hai prompt cạnh nhau (di chuột để xem hiệu quả):
- Prompt chung chung kiểu "tóm tắt cuộc hội thoại đến giờ": tạo ra bản tóm tắt tổng quát, có thể rơi mất trạng thái quan trọng của tác vụ: file nào đã sửa, quyết định nào đã chốt ở ngã rẽ, lỗi nào đã gặp và đã xử lý ra sao.
- Prompt chỉ rõ cần giữ **mọi đường dẫn file đã sửa, mọi quyết định đã chốt, mọi lỗi gặp phải cùng cách giải quyết**: tạo ra bản tóm tắt agent dùng được.

Kết luận của bài: đây không phải trường hợp hiếm. Mất trạng thái quan trọng vì bộ tóm tắt viết thiếu cụ thể là một trong những nguồn lỗi phổ biến nhất của agent chạy nhiều phiên.

## 7. Subagent handoffs: quản lý tác vụ đường dài

- Khi tác vụ quá lớn cho một window, **tăng window không phải lời giải**. Lời giải là chia nhỏ tác vụ và chỉ chuyển phần ngữ cảnh liên quan cho từng subagent.
- Subagent nhận: một tác vụ có phạm vi rõ, **lượng ngữ cảnh tối thiểu** cần thiết, kết quả của các bước trước có liên quan trực tiếp, các tool cần dùng và **điều kiện kết thúc rõ ràng**. Agent cha thu kết quả về.
- Lợi ích: chi phí mỗi lượt thấp và tác vụ đường dài khả thi.
- Giống pruning và compaction, handoff có chi phí triển khai, nên chỉ dùng khi chi phí context là ràng buộc thật. Prompt đơn lượt hoặc workflow ngắn không cần.

Hộp tổng kết của bài:
- **Xử lý tốt:** phiên agent nhiều bước vượt ngân sách token và cần được phân rã. Nên thiết kế từ giai đoạn kiến trúc, không vá thêm như một bản sửa production.
- **Nên dùng cách khác:** pipeline không bao giờ tiến gần giới hạn window. Hãy đo mức dùng token thực tế so với giới hạn context của model trước khi thêm chi phí quản lý.

## 8. Forward pointer (gợi ý sang màn sau)

Các chiến lược ở trên giả định bạn biết ngân sách context đang chịu áp lực và đang chọn công cụ để xử lý. Điểm then chốt là **không biết áp lực tồn tại cho đến khi phiên hỏng**. Một workload có thể qua mọi test lúc phát triển rồi hỏng ở production chỉ vì output tool lớn hơn, phiên dài hơn, và window từng chứa 20 lượt giờ đầy ở lượt 8. Màn kế tiếp dẫn một postmortem về agent chạy tốt trên dữ liệu test rồi chạm trần khi tài liệu thật bắt đầu chảy vào.

## Phân tích thêm (của Claude, không có trong bài)

- **Cách nhớ bốn chiến lược:** xếp theo mức "mất bao nhiêu" từ nhẹ đến nặng thì là *subagent handoff* (mất quá trình, giữ kết quả) → *compaction* (mất chi tiết) → *pruning* (mất công việc sau điểm quay lại) → *clearing* (mất tất cả). Cách sắp này là của tôi để dễ ôn, không phải thứ tự trong bài.
- **Hai chiến lược dễ bị hỏi lẫn trong đề thi:** pruning khác clearing ở chỗ pruning giữ phần đầu hội thoại còn clearing bỏ hết. Clearing buộc bạn phải có nơi lưu bền vững như `CLAUDE.md` nếu cần nhớ xuyên phiên.
- **Mức độ chắc chắn:** các nhận định trong mục 1-8 bám theo nội dung bài. Riêng chi tiết về tên các tầng model, con số "ba đến năm lần" và "lượt 8 so với lượt 50" là số liệu minh họa trong bài, không phải số đo tôi tự xác nhận. Những chi tiết phụ thuộc phiên bản (danh sách model, trạng thái beta của server-side compaction, con số hiệu năng agentic search) nên được đối chiếu lại với tài liệu chính thức trước khi dùng cho đề thi hoặc cho hệ thống thật.
- **Một lỗ hổng nhỏ trong ghi chú này:** hai thao tác cuộn trang bị quá thời gian nên tôi không đọc lại được vài câu ở phần mở đầu của màn, giữa đoạn "mọi kết quả tool được thêm vào window" và đoạn "agent hoặc nén hoặc đứng khựng". Hai đoạn đó nối liền ý với nhau nên ghi chú không bị mất ý chính, nhưng nếu bạn thấy câu nào bị thiếu thì mở lại phần đầu màn để đối chiếu.
