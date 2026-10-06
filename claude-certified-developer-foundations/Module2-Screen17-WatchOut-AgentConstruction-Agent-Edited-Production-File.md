---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Agent construction
screen: 17 of 29
screen_type: WATCH OUT (lỗi thường gặp / postmortem, khoảng 5 phút)
topic: Agent construction
screen_title: The agent that edited a production file
related_screens: Screen 16 (Teaching - HITL và checklist nối vòng lặp)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần đầu bám đủ nội dung màn gốc, phần "Tóm tắt, ELI5 và ví dụ" ở cuối là phần Claude thêm
---

# [Module 2 · Screen 17 · WATCH OUT] Agent đã sửa một file production

## Bối cảnh (phần "Setup")

Agent chạy ngon end-to-end khi test vì môi trường test **tha thứ** còn production thì không. Agent có cùng tool, cùng vòng lặp, cùng system prompt, nhưng **thiếu checkpoint HITL**, vì lúc test chưa từng xuất hiện tình huống nào cần đến nó.

## Case study: agent sửa file, test trong thư mục tạm, triển khai lên môi trường khách hàng

**Thiết kế.** Một developer xây agent có thể đọc, sửa và ghi file cấu hình. System prompt cấp 3 tool: `read_file`, `write_file`, `validate_config`. Vòng lặp khá đơn giản: sau mỗi lần ghi, agent chạy lại `validate_config`; nếu config vẫn lỗi thì agent chỉnh và ghi lại, **tối đa mười vòng** rồi dừng.

**Lúc test.** Agent được test trên một thư mục tạm chứa bản sao của config đích. Mọi ca test đều đúng, thường hội tụ về config hợp lệ sau 2-3 vòng.

**Khi triển khai lên môi trường khách hàng.** Agent nhận ra đúng một tham số nằm ngoài phạm vi cho phép. Nó đề xuất sửa, gọi `write_file`, chạy lại `validate_config` và nhận pass. Vòng lặp kết thúc gọn sau **một** vòng, đúng như thiết kế. Giới hạn mười vòng không bao giờ bị chạm vì không cần. Bài chốt: thiết kế vòng lặp là đúng, **vấn đề nằm ở điều kiện thoát**.

**Sự cố.** Tham số được sửa là một **rate limit mà ứng dụng của khách hàng dựa vào**. `validate_config` chỉ kiểm giá trị mới có nằm trong khoảng cho phép của schema hay không (và có, nằm trong khoảng). Cái nó không kiểm, và cũng không được thiết kế để kiểm, là **các hệ thống phía dưới có đang phụ thuộc vào giá trị cũ hay không**. Trong vài phút sau lần ghi, ứng dụng của khách hàng bắt đầu lỗi vì request bị throttle ở mức mà nó không được xây dựng để chịu.

**Chẩn đoán.** Vòng lặp của agent làm đúng những gì developer yêu cầu: sửa, validate, thoát khi validate pass. Lỗi không ở vòng lặp mà ở chỗ **điều kiện thoát (`validate_config` trả pass) chỉ có phạm vi là file đang sửa**, và giữa "validation pass trên file này" với "ghi vào môi trường khách hàng" **không có checkpoint nào**.

**Phần còn thiếu.** Một checkpoint trong thiết kế vòng lặp: trước khi lần gọi `write_file` đầu tiên chạm vào config sống của khách hàng, hãy **tạm dừng và đưa thay đổi đề xuất ra cho con người duyệt**. Trong thực tế, vòng lặp cần một **nhánh tường minh** giữa trạng thái "thay đổi đề xuất đã sẵn sàng" và "thay đổi đã được ghi", một trạng thái mà developer chưa từng thêm vì lúc test không có ca nào đòi hỏi nó.

| | Test (thư mục tạm) | Môi trường khách hàng |
|---|---|---|
| Tool, vòng lặp, system prompt | Như nhau | Như nhau |
| Số vòng | 2-3 | 1 |
| Điều kiện thoát | `validate_config` pass | `validate_config` pass |
| Ghi sai thì sao | Không ảnh hưởng gì | Ứng dụng khách hàng lỗi sau vài phút |
| Checkpoint HITL | Không có, không ai thấy thiếu | Không có, và đây là chỗ hỏng |

## Điều cần nhớ (phần "What to watch out for")

1. **Đây là một câu hỏi về quyền hạn chưa bao giờ được đặt ra lúc thiết kế.** Agent có quyền ghi vì nhiệm vụ là sửa file, và nhiệm vụ đó hợp lệ. Cái nhóm bỏ sót là **khoảng cách giữa agent đề xuất một thay đổi và agent thực thi thay đổi đó**. Trong môi trường test dùng một lần, khoảng cách này không bao giờ lộ ra vì không có "lần ghi sai" nào gây hậu quả khi test; production thì khác.
2. **Câu hỏi thiết kế bị bỏ sót:** "Kết quả xấu nhất là gì nếu `write_file` chạy mà không có người kiểm tra?" Câu trả lời cho câu hỏi này quyết định có cần checkpoint HITL trước khi tool được phép chạy hay không.
3. **Quy tắc:** nếu một tool có thể thực hiện hành động **không hoàn tác được** ở production thì nó cần một checkpoint trước khi chạy. Hãy ghi ràng buộc này ngay **lúc thiết kế, khi đang xác định phạm vi bề mặt tool**, chứ không đợi sau sự cố đầu tiên.

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt một câu:** `validate_config` pass chỉ nói "file hợp lệ", không nói "ghi vào hệ thống thật là an toàn", nên cần một chốt chặn con người trước hành động không hoàn tác.

**ELI5:** bạn nhờ cậu bé chỉnh nút âm lượng cái loa. Ở nhà tập chỉnh sai cũng chẳng sao. Đến nhà hàng, cậu vặn nhỏ nút xuống mức vẫn "đúng vạch cho phép" rồi báo "xong!", nhưng cả buổi tiệc đang cần nhạc nền to. Cái thiếu là câu hỏi trước khi vặn: "Chú ơi, con chỉnh thế này được không?"

**Snippet 1: chốt chặn trong tool ghi**

```python
if env in PROTECTED_ENVS:
    decision = approver(path=path, diff=make_diff(old, new))
    if decision != "approve":
        return {"status": "rejected_by_human"}
open(path, "w").write(new)
```

Dùng để: chỉ production mới phải chờ người duyệt, test vẫn ghi thẳng. Chốt chặn nằm trước dòng ghi, không nằm ở `validate_config`.

**Snippet 2: điều kiện thoát của vòng lặp**

```python
for _ in range(MAX_ITERATIONS):
    resp = client.messages.create(...)
    if resp.stop_reason != "tool_use":
        break
```

Dùng để: dừng khi Claude không gọi tool nữa, có giới hạn số vòng. Khi người từ chối, agent nhận `rejected_by_human` làm tool_result và đổi cách làm, vòng lặp không vỡ.
