---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Agent memory
screen: 19 of 29
screen_type: TEACHING (bài giảng, khoảng 8 phút)
topic: Agent memory
screen_title: Choosing the right scope for state that survives sessions
related_screens: Screen 13 (context engineering, compaction), Screen 16 (settingSources của Agent SDK)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần cuối "Tóm tắt, ELI5 và ví dụ" là phần Claude thêm
---

# [Module 2 · Screen 19 · TEACHING] Chọn scope cho state sống qua nhiều session

## Mở đầu

Agent ở phần trước chạy đúng **trong một session**. Cái nó không làm được là nhớ gì khi session kết thúc. **Memory scope** là cách bạn quyết định agent cần biết gì ở đầu session kế tiếp, và mang kiến thức đó đi tiếp tốn bao nhiêu.

## Các mẫu hình agent và vị trí của memory

Ngoài memory scope, blueprint gom nhiều mẫu hình agent vào cùng mục tiêu, và phần lớn bạn đã gặp:
- **Vòng lặp tool-use** (model gọi tool, đọc kết quả, tiếp tục): mẫu cốt lõi của cụm tool-use và agent construction.
- **Phân rã tác vụ nhiều bước:** chia mục tiêu thành các subtask có thứ tự.
- **Tách lập kế hoạch và thực thi:** quyết định kế hoạch tách khỏi việc thực hiện, cùng kiểu với kiểm tra HITL sau bước lập kế hoạch.
- **Memory scope** (phần này): mẫu quyết định state nào sống sót khi vòng lặp kết thúc.

Chọn sai scope có **hai kiểu hỏng kéo ngược chiều nhau**:
- Quá nhiều state trong context: mỗi lần gọi API model phải đọc lại toàn bộ hội thoại, chi phí tăng theo độ dài session.
- Quá ít state trong lưu trữ bền: agent mất trí nhớ giữa các session, vì thứ gì không được ghi ra thì biến mất khi hội thoại kết thúc.

## Bảng bốn scope

| Scope | Cái gì tồn tại | Chi phí | Dùng khi | Mất gì |
|---|---|---|---|---|
| **In-context memory** | State nằm trong hội thoại đang chạy, sống qua các lượt trong một session | Không tốn truy xuất; chi phí token tăng theo hội thoại | Session ngắn, mọi state vừa trong context window, không gì phải mang qua lần khởi động lại | Mọi thứ khi session kết thúc; một lệnh clear hay session mới xóa sạch |
| **External storage** | State ghi vào database, đọc lại khi bắt đầu session hoặc theo yêu cầu | Mỗi lần gọi DB thêm độ trễ truy xuất, và bạn gánh công kỹ thuật viết logic đọc/ghi | State phải sống qua các session, chuyển giữa người dùng, hoặc chia sẻ giữa nhiều instance agent | Không mất gì phía lưu trữ; cái giá là độ trễ mỗi lần gọi và độ phức tạp triển khai |
| **Summarized memory** | Bản tóm tắt cô đọng của hội thoại trước được tạo ra và chèn vào đầu session sau | Rẻ token hơn phát lại toàn bộ lịch sử, nhưng bước tóm tắt làm rơi chi tiết của bản gốc | Agent hội thoại chạy dài mà toàn bộ lịch sử sẽ vượt ngân sách context trước khi hội thoại xong | Mọi chi tiết bộ tóm tắt không giữ; agent chỉ thấy thứ prompt tóm tắt chọn giữ |
| **No persistent memory (stateless)** | Không gì cả, mỗi session độc lập | Không tốn gì vì không có gì để truy xuất hay lưu | Agent thực thi tác vụ rồi đóng, hoặc pipeline mà mỗi session độc lập theo thiết kế | Toàn bộ ngữ cảnh trước; nếu việc theo sau phụ thuộc điều gì từ session trước, agent không có cách nào lấy lại |

## Chọn memory scope lúc thiết kế

- Cách agent nhớ tương tác trước **thuộc về giai đoạn thiết kế**, không phải thứ để refactor sau khi lên production.
- Agent giúp cùng một người dùng qua nhiều ngày cần mang state giữa các session: lưu tóm tắt hoặc toàn bộ lịch sử **bên ngoài context window** để session sau đọc lại. Agent nhận một việc, làm xong rồi đóng thì không có session trước để nhớ, nên chạy stateless.
- **Con đường mặc định nghe có vẻ ổn lúc đầu:** lưu toàn bộ hội thoại trong mảng messages, gửi mỗi lần gọi API, prototype chạy tốt một thời gian. Rắc rối đến về sau: chi phí token tăng theo mỗi lượt, độ trễ tăng khi context đầy, và một session dài chạm giới hạn cứng thì agent ngừng trả lời. Lúc đó bạn phải refactor: kéo state hội thoại ra khỏi context sống, đưa vào lưu trữ ngoài và chỉ thêm vào cái mỗi lượt cần.
- Bản thân việc refactor mang tính cơ học (vài trăm dòng code và một database team đã có). **Cái giá là thời điểm:** nó diễn ra dưới áp lực production, thường sát deadline, và mỗi giờ tái cấu trúc memory là một giờ không dành cho việc agent phải làm. Quyết định lúc thiết kế rẻ, quyết định lúc phải refactor đắt hơn.

Hộp tổng kết của bài:
- **Xử lý tốt:** scope memory khớp với tác vụ ngay từ thiết kế. Dùng external storage khi agent tiếp tục một mạch qua các session, stateless khi mỗi job tự đủ, in-context khi session ngắn và không cần sống qua khởi động lại.
- **Thêm chi phí hoặc phức tạp:** external storage thêm độ trễ truy xuất và logic đọc/ghi. Summarized memory phụ thuộc prompt tóm tắt được chỉ định tốt; thiếu nó thì state quan trọng của tác vụ bị bỏ ở mỗi lần nén. Không cách nào miễn phí.
- **Dùng cách khác khi:** bạn đang giữ mọi state trong context với giả định window đủ lớn. Chi phí token tăng theo từng lượt vì toàn bộ context được gửi mỗi lần gọi. Không có caching hoặc compaction, session dài tích lũy chi phí nhanh hơn dự kiến nếu chỉ đo các lượt đầu. Hãy đo mức dùng token thực tế của session so với giới hạn window trước khi cam kết.

## Skills: bộ chỉ dẫn tái sử dụng, nạp theo yêu cầu

Bảng memory trên nói cách agent mang **state** qua các session. Còn một vấn đề liên quan nhưng khác: mang **chỉ dẫn lặp lại** qua các tác vụ mà không phải trả phí nhét chúng vào mọi session.

- Mẫu hình cho việc này là **Skill**: một file markdown tái sử dụng dạy Claude xử lý một loại tác vụ cụ thể một lần. Claude **tự nạp Skill khi yêu cầu khớp với mô tả** của nó. Chỉ dẫn nằm trên đĩa đến khi cần, không thường trú trong mọi hội thoại.
- Một Skill nằm trong file `SKILL.md` trong một thư mục xác định. File có hai phần: khối **frontmatter** (tên và mô tả) và phần chỉ dẫn bên dưới. **Mô tả là tiêu chí khớp.** Khi bạn gửi yêu cầu, Claude đọc tên và mô tả của mọi Skill có sẵn, so với tin nhắn của bạn, và chỉ nạp chỉ dẫn đầy đủ khi có khớp. Chỉ dẫn không liên quan đến yêu cầu hiện tại thì không bao giờ vào context window.
- **Điểm tương phản với các mẫu memory:** in-context memory luôn hiện diện và lớn dần theo từng lượt. Hành vi của `CLAUDE.md` tùy nơi bạn chạy Claude Code: ở Claude Code CLI, file `CLAUDE.md` nạp vào mọi session bất kể tác vụ gì. Ở Agent SDK, việc nạp các thiết lập filesystem (gồm `CLAUDE.md`) do cấu hình **`settingSources`** điều khiển. Đừng dựa vào mặc định: hãy đặt tường minh theo nguồn bạn muốn và đối chiếu hành vi mặc định hiện hành với tài liệu tham chiếu Agent SDK lúc build. Còn Skill chỉ nạp khi tác vụ cần, ở cả hai môi trường. Với chỉ dẫn áp dụng cho tác vụ lặp lại cụ thể chứ không phải mọi session, Skill là mẫu có overhead thấp hơn hai lựa chọn kia.

### Skill vs CLAUDE.md vs chỉ dẫn in-context

| Mẫu | Khi nạp | Chi phí context | Hợp nhất cho |
|---|---|---|---|
| **Skill (SKILL.md)** | Theo yêu cầu, khi request khớp mô tả của skill | Thấp: chỉ tên và mô tả nạp lúc khởi động; nội dung đầy đủ chỉ nạp khi khớp | Chuyên môn riêng của tác vụ không nên làm phình các session không cần, ví dụ định dạng đầu ra chuyên ngành, checklist review chuyên biệt, workflow áp dụng cho một tập con tác vụ |
| **CLAUDE.md** | Mọi session, vô điều kiện | Overhead cố định mỗi session bất kể tác vụ | Chuẩn dự án luôn bật áp dụng cho mọi thứ, ví dụ quy ước code team đã chuẩn hóa, quy tắc định dạng đầu ra dự án yêu cầu, ràng buộc đúng xuyên suốt codebase |
| **Chỉ dẫn in-context** | Hiện diện ở mọi lượt trong session đó | Tăng theo độ dài session; không sống qua khi session kết thúc | Session ngắn mà toàn bộ lịch sử vừa trong window và không gì cần tồn tại lâu, ví dụ việc khám phá một lần, tác vụ gói trong một hội thoại |

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt**

- Bốn scope: in-context (mất khi hết session), external storage (bền, tốn độ trễ và code), summarized (rẻ nhưng rơi chi tiết, cần prompt tóm tắt cụ thể), stateless (không gì, hợp job tự đủ).
- Quyết định lúc **thiết kế**, vì refactor lúc production vừa tốn vừa trễ.
- Chỉ dẫn lặp lại: **Skill** nạp theo yêu cầu (rẻ), **CLAUDE.md** nạp mọi session (cố định), **in-context** mất khi hết session. Với Agent SDK, đặt `settingSources` tường minh.

**ELI5**

In-context là ghi nháp lên bảng trắng, hết buổi họp thì lau sạch. External storage là sổ tay cất tủ, hôm sau mở ra đọc lại (phải đi lấy). Summarized là tờ ghi chú gạch đầu dòng của buổi họp, gọn nhưng thiếu chi tiết nếu người ghi bỏ sót. Stateless là cuộc gọi hỗ trợ mà mỗi lần gọi đều gặp nhân viên mới không biết gì. Skill là cuốn sổ hướng dẫn xếp trên kệ, chỉ lấy xuống khi gặp đúng loại việc; CLAUDE.md là tờ nội quy dán ngay cửa, ai vào cũng đọc.

**Ví dụ khi implement**

*Snippet 1: external storage, nạp state khi bắt đầu session và ghi lại khi kết thúc*

```python
state = db.get(user_id) or {"summary": ""}
messages = [{"role": "user", "content": f"Ngữ cảnh trước: {state['summary']}\n\n{user_request}"}]
# ... chạy vòng lặp agent ...
db.put(user_id, {"summary": new_summary})
```

Dùng để: agent nhớ người dùng qua nhiều ngày mà context mỗi session chỉ mang phần tóm tắt cần thiết.

*Snippet 2: frontmatter của một Skill (mô tả là tiêu chí khớp)*

```markdown
---
name: review-checklist
description: Dùng khi người dùng yêu cầu review pull request theo checklist bảo mật của team
---
(các chỉ dẫn chi tiết nằm ở đây, chỉ nạp khi request khớp mô tả)
```

Dùng để: chỉ dẫn chuyên biệt không chiếm context của các session không liên quan.
