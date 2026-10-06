---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Agent memory
screen: 20 of 29
screen_type: WATCH OUT (lỗi thường gặp / postmortem, khoảng 2 phút)
topic: Agent memory
screen_title: The agent that filled the window on session four
related_screens: Screen 19 (Teaching - memory scope), Screen 14 (Watch Out - ngân sách context chưa đo với dữ liệu thật)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần cuối "Tóm tắt, ELI5 và ví dụ" là phần Claude thêm
---

# [Module 2 · Screen 20 · WATCH OUT] Agent làm đầy window ở session thứ tư

## Bối cảnh (phần "Setup")

Agent chạy hoàn hảo lúc phát triển vì bạn chạy nó trong **một session dài liên tục**: context window không bao giờ đầy nên in-context memory giữ được mọi thứ. Nhưng production chạy **nhiều session ngắn hơn, nhiều lượt hơn, trải qua nhiều ngày**, và window đầy ở session thứ tư.

## Postmortem: state in-context phình lên cho đến khi chạm trần window

1. **Thiết kế:** một agent hỗ trợ kỹ sư support xử lý các ca escalation kéo dài.
2. **Lúc dev:** chạy các session liên tục 10-15 lượt. State in-context giữ đúng toàn bộ lịch sử. Developer ra mắt mà **không đo mức dùng token mỗi session**.
3. **Ở production:** mỗi session ngắn hơn, nhưng **state tích lũy qua các session**. Đến session thứ tư, lịch sử in-context được chèn vào đã vượt **40.000 token** trước khi agent xử lý được một tool call nào. Cộng với system prompt và schema các tool đã đăng ký, hơn **45.000 token** ngân sách context đã bị tiêu trước lượt làm việc đầu tiên của session.
4. **Hệ quả:** khi các tool call tích lũy trong session, phần ngân sách còn lại cạn trước khi agent phân tích xong. Agent bắt đầu trả về **kết quả dở dang**, một triệu chứng ban đầu trông như **lỗi chọn tool** chứ không phải vấn đề kiến trúc memory.
5. **Cách sửa:** refactor sang **external storage** trong một giờ: kéo lịch sử session tích lũy ra khỏi context sống, lưu vào database, và chỉ chèn **tập con liên quan** lúc bắt đầu session.
6. **Cái giá:** việc refactor dưới áp lực production tốn **lâu hơn đáng kể** so với nếu làm ở giai đoạn thiết kế. Tầng lưu trữ, logic truy xuất và quản lý session đều cần những quyết định lẽ ra phải được đưa ra **trước lần triển khai đầu tiên**.

## Điều cần nhớ (phần "What to watch out for")

Lúc dev dùng một session dài, production dùng nhiều session ngắn với state tích lũy. Đó là hai hình dạng khác nhau và in-context memory xử lý chúng khác nhau. **Hãy đo kích thước state dự kiến mỗi session (lịch sử + system prompt + schema tool) so với giới hạn context** trước khi chọn in-context làm mặc định.

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt**

- Dev: một session dài, không chạm trần. Production: nhiều session ngắn, state cộng dồn, session 4 đã tốn hơn 45k token trước lượt đầu.
- Triệu chứng giống lỗi chọn tool, nhưng gốc là chọn sai scope memory.
- Sửa: đưa lịch sử ra external storage, chỉ nạp phần liên quan đầu session. Làm lúc thiết kế sẽ rẻ hơn nhiều.
- Phòng ngừa: đo (lịch sử + system prompt + schema tool) so với giới hạn context trước khi chọn in-context.

**ELI5**

Bạn nhét tất cả thư cũ của khách vào một cái balo để mang theo mỗi lần đi làm. Một ngày đi thì balo nhẹ. Đến ngày thứ tư balo nặng đến mức không còn chỗ chứa đồ nghề, nên bạn làm việc dở dang, và mọi người tưởng tay nghề bạn kém. Thực ra chỉ cần để thư cũ ở tủ hồ sơ, mỗi lần chỉ mang vài lá liên quan.

**Ví dụ khi implement**

*Snippet 1: đo trước khi chọn in-context làm mặc định*

```python
est = client.messages.count_tokens(
    model=MODEL, system=SYSTEM, tools=TOOLS,
    messages=load_history(user_id) + [{"role": "user", "content": new_request}],
).input_tokens
if est > CONTEXT_BUDGET * 0.5:
    use_external_storage = True   # đừng nhét cả lịch sử vào context
```

Dùng để: phát hiện sớm state tích lũy sẽ ăn hết ngân sách, trước khi lên production.

*Snippet 2: chỉ chèn tập con liên quan của lịch sử*

```python
relevant = db.query(user_id, topic=new_request, limit=5)
messages = [{"role": "user",
             "content": f"Ngữ cảnh liên quan:\n{relevant}\n\n{new_request}"}]
```

Dùng để: mỗi session bắt đầu với lượng context nhỏ và ổn định, bất kể đã có bao nhiêu session trước.
