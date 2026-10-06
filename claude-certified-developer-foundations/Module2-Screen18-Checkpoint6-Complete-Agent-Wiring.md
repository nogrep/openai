---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Agent construction
screen: 18 of 29
screen_type: CHECKPOINT (bài tập tự kiểm tra, khoảng 4 phút)
checkpoint_number: 6
topic: Agent construction
screen_title: Checkpoint 6 - Complete the agent wiring
related_screens: Screen 16 (Teaching - bốn bước nối vòng lặp, bảng điểm chèn HITL, over/under-tooling), Screen 17 (Watch Out - agent sửa file production)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần giải thích từng dòng code là của Claude. Tôi chỉ đọc đáp án mẫu, không bấm nút tự chấm nào trên trang
---

# [Module 2 · Screen 18 · CHECKPOINT 6] Hoàn thiện agent wiring

## Đề bài

Bài đưa một bản triển khai agent dở dang có **hai chỗ trống**. Người học phải viết:
1. **Gap 1:** phần `description` cho tool `update_record`.
2. **Gap 2:** code **checkpoint HITL** trước khi thực thi `update_record`.

### Bộ khung đề cho sẵn (mô tả lại)

- Danh sách `tools` có hai tool:
  - `read_record`: có sẵn mô tả "dùng để đọc bản ghi khách hàng theo `customer_id`"; tham số `customer_id` (string, bắt buộc).
  - `update_record`: **thiếu `description` (chỗ trống 1)**; tham số `customer_id`, `field`, `new_value` (đều string, đều bắt buộc).
- Hàm `run_agent_loop(user_request)`:
  - Khởi tạo `messages` bằng lượt người dùng.
  - `while True`: gọi `client.messages.create(model, max_tokens=4096, tools, messages)`.
  - Nếu `stop_reason == "end_turn"` thì trả về `response` (điều kiện thoát).
  - Nếu `stop_reason == "tool_use"`: append lượt assistant (`response.content`) vào `messages`, duyệt từng block; với block `tool_use` thì **[chỗ trống 2: chèn checkpoint HITL trước khi thực thi `update_record`]**, rồi `execute_tool(block.name, block.input)`, gom thành `tool_result` (có `tool_use_id`), cuối cùng append một lượt `user` chứa toàn bộ `tool_results`.

## Đáp án mẫu

### Gap 1: description cho `update_record`

Ý của đáp án (diễn giải): "Dùng để cập nhật **một trường cụ thể** trên bản ghi khách hàng. **Chỉ gọi tool này sau khi** một lần `read_record` đã xác nhận giá trị hiện tại và thay đổi đề xuất đã được xem xét. **Không dùng** cho cập nhật hàng loạt hay thay đổi schema."

Bài giải thích vì sao đây là cách viết tốt: mô tả mang tính **hạn chế (restrictive)**, nói cho agent biết (a) khi nào **không** được gọi, (b) điều gì phải đúng **trước** khi gọi, (c) tool **không bao giờ** dùng cho việc gì, bằng ngôn ngữ mà model có thể dùng để định tuyến. Một mô tả trơn (kiểu "cập nhật bản ghi") sẽ để agent gọi `update_record` mỗi khi nó suy ra cần cập nhật, kể cả **trước khi đọc giá trị hiện tại** hoặc trên **trường mà người vận hành không có ý muốn đổi**.

### Gap 2: code checkpoint HITL

```python
if block.type == "tool_use":
    if block.name == "update_record":
        print(f"Proposed update, customer_id: {block.input['customer_id']}, "
              f"field: {block.input['field']}, new_value: {block.input['new_value']}")
        approval = input("Approve this update? (yes/no): ").strip().lower()
        if approval != "yes":
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": "Update rejected by operator.",
            })
            continue
    result = execute_tool(block.name, block.input)
```

Bài chốt: checkpoint nằm **bên trong vòng lặp** và **gác theo tên tool**, nên `read_record` đi thẳng qua, còn `update_record` phải dừng chờ duyệt. Một lần duyệt chung ở đầu thì không chặn được **một thao tác cập nhật cụ thể mà model chưa đề xuất**. Và duyệt **sau khi** `execute_tool` chạy thì việc không hoàn tác đã xong rồi.

Hai nút cuối trang ("Both gaps match · pass" / "Missed one · retry") là nút tự chấm; tôi không bấm.

## Giải thích kỹ code đáp án

Đi từng dòng theo đúng thứ tự thực thi.

1. **`if block.type == "tool_use":`**
   Một lượt trả lời của Claude là danh sách nhiều block (text, tool_use...). Chỉ block `tool_use` mới là yêu cầu chạy tool; block text thì bỏ qua. Đây là chỗ duy nhất vòng lặp "thực thi", nên cũng là chỗ hợp lý để đặt chốt chặn.

2. **`if block.name == "update_record":`**
   Đây là **gác theo tên tool**. Chỉ tool có thể ghi mới bị chặn. `read_record` không có nhánh này nên chạy thẳng xuống `execute_tool`. Nếu gác tất cả tool thì người duyệt phải bấm yes cho cả những lần đọc vô hại, nhanh chóng trở thành bấm bừa (approval fatigue).

3. **`print(f"Proposed update, customer_id: ..., field: ..., new_value: ...")`**
   Đưa cho người duyệt **đúng thứ sắp được thực thi**: bản ghi nào, trường nào, giá trị mới là gì. Người duyệt cần thấy tham số **model đã đề xuất** (`block.input`), không phải một mô tả chung chung. Không có bước này thì "duyệt" chỉ là hình thức.

4. **`approval = input("Approve this update? (yes/no): ").strip().lower()`**
   Chặn luồng chương trình tại đây (tạm dừng agent) cho đến khi con người trả lời. `.strip().lower()` chuẩn hóa chuỗi để "  Yes " cũng hiểu là "yes".

5. **`if approval != "yes":`**
   Chiều an toàn: **chỉ đúng "yes" mới cho qua**, mọi thứ khác (no, rỗng, gõ nhầm) đều bị từ chối. Mặc định là từ chối, không phải mặc định cho phép.

6. **`tool_results.append({"type": "tool_result", "tool_use_id": block.id, "content": "Update rejected by operator."})`**
   Dù từ chối, **vẫn phải trả một `tool_result`** cho block này, gắn đúng `tool_use_id`. Lý do: API yêu cầu mỗi `tool_use` có một `tool_result` tương ứng ở lượt kế (xem checklist Screen 16, mục "mọi tool-use block trong cùng một lượt phải được giải quyết"). Nếu bỏ qua, request sau bị từ chối vì cặp tool_use / tool_result không khớp. Nội dung "rejected" cũng cho Claude biết chuyện gì đã xảy ra để nó đổi hướng (hỏi lại người dùng, đề xuất cách khác) thay vì tưởng đã cập nhật xong.

7. **`continue`**
   Bỏ qua phần còn lại của vòng lặp `for` cho block này, tức là **không gọi `execute_tool`**. Đây là dòng thực sự "chặn" việc ghi. Các block khác trong cùng lượt vẫn được xử lý tiếp.

8. **`result = execute_tool(block.name, block.input)`**
   Chỉ chạy tới đây khi (a) tool không phải `update_record`, hoặc (b) là `update_record` và người duyệt đã nói yes. Thứ tự "duyệt trước, chạy sau" là toàn bộ ý nghĩa của checkpoint.

9. **Phần có sẵn phía sau:** kết quả được gói thành `tool_result` rồi cả danh sách `tool_results` được append thành **một** lượt `user`. Dù có block bị từ chối hay được chạy, mọi block của lượt đều có kết quả đi kèm.

Các lỗi hay gặp nếu viết khác đi (để so với đáp án):
- Đặt `input()` **sau** `execute_tool`: việc ghi đã xảy ra, hỏi lại là vô nghĩa.
- Hỏi duyệt **một lần ở đầu** hàm: model chưa đề xuất update cụ thể nào, nên không có gì để duyệt.
- Từ chối nhưng **không trả `tool_result`**: lượt sau hỏng vì tool_use mồ côi.
- Từ chối nhưng **quên `continue`**: tool vẫn chạy.
- Gác mọi tool thay vì theo tên: người duyệt mệt, nhanh chóng bấm yes cho tất cả.

## Mối liên hệ với các màn trước

Checkpoint này là phần code của chính bài học ở Screen 16 (bước 3 xử lý tool-use loop, điểm HITL "trước tool call phá hủy") và là cách sửa cho sự cố ở Screen 17 (agent ghi file production mà không có chốt chặn).

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt**

- Gap 1: mô tả tool phải **hạn chế**: khi nào không gọi, điều kiện trước khi gọi, không dùng cho việc gì.
- Gap 2: checkpoint nằm **trong vòng lặp**, **gác theo tên tool**, hiện đúng tham số đề xuất, **chỉ "yes" mới chạy**, từ chối thì **vẫn trả `tool_result`** và `continue`.
- Duyệt phải diễn ra **trước** `execute_tool`, không phải sau.

**ELI5**

Nhân viên thu ngân (agent) được phép xem hóa đơn thoải mái (read), nhưng muốn **hoàn tiền** (update) thì phải gọi quản lý. Quản lý được nghe đúng "hoàn bao nhiêu, cho ai" (print đề xuất), rồi nói "đồng ý" hoặc không. Nếu không, thu ngân vẫn phải ghi vào sổ "bị từ chối" để khách và sổ sách khớp nhau (tool_result), rồi mới sang việc khác (continue). Hỏi quản lý **sau khi** đã hoàn tiền thì vô nghĩa.

**Ví dụ khi implement**

*Snippet 1: gác theo danh sách tool nguy hiểm thay vì một tên cứng*

```python
NEEDS_APPROVAL = {"update_record", "delete_record", "send_email"}

if block.name in NEEDS_APPROVAL:
    ...  # hỏi duyệt như đáp án
```

Dùng để: mở rộng khi có thêm tool ghi mà không phải sửa nhiều chỗ; tool đọc vẫn đi thẳng.

*Snippet 2: duyệt bất đồng bộ trong ứng dụng web (thay cho `input()`)*

```python
approval = await approvals.request(  # đẩy lên UI, chờ người bấm
    tool=block.name, args=block.input, timeout=300
)
if approval != "approved":
    results.append({"type": "tool_result", "tool_use_id": block.id,
                    "content": "Update rejected or timed out."})
    continue
```

Dùng để: bản `input()` chỉ hợp với CLI. Ứng dụng thật cần hàng đợi duyệt và timeout; hết giờ coi như từ chối (mặc định an toàn).
