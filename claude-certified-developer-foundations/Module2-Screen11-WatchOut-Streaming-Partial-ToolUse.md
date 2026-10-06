---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
screen: 11 of 29
screen_type: WATCH OUT (lỗi thường gặp / postmortem)
topic: Streaming responses
screen_title: The stream that left a half-written tool call in the history
related_screens: Screen 10 (Teaching - Streaming responses and handling partial output without corrupting state)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học
---

# [Module 2 · Screen 11 · WATCH OUT] Streaming: tool call viết dở bị lưu vào history

## Một câu tóm tắt

**Stream kết thúc không có nghĩa là message đã hoàn tất.** Chỉ `message_stop` mới xác nhận message đã trọn vẹn.

## Bối cảnh (phần "Setup")

Một response streaming có thể trông hoàn toàn bình thường trên màn hình: text hiển thị, người dùng thấy câu trả lời, handler đã thêm lượt đó vào history. Nhưng nếu stream thực ra đã rơi giữa chừng, thì tool_use được lưu trong history **thiếu một nửa input**. Request kế tiếp sẽ fail ở bước validation, và lỗi báo ra ở **lượt sau**, không phải ở stream đã gây ra nó. Vì vậy người debug rất dễ nhìn nhầm chỗ.

## Case study (Postmortem): tool_use dở dang bị commit sau khi stream rớt

Diễn biến theo thời gian:

1. **Thiết kế:** một agent dùng streaming để người vận hành xem response sinh ra theo thời gian thực. Handler gom các event `content_block_delta`, và khi vòng đọc stream kết thúc thì thêm lượt assistant vào history.
2. **Lúc test:** chạy trên kết nối local nhanh nên stream luôn chạy đến cuối. Vòng đọc luôn kết thúc tại `message_stop`, nên các lượt lưu vào history lúc nào cũng hoàn chỉnh. Tức là test **không bao giờ chạm được** nhánh lỗi.
3. **Trên production:** một lần mạng chập chờn làm stream đứt **sau khi** block tool_use đã mở và nhận được một phần JSON input, nhưng **trước** `content_block_stop`.
4. **Handler làm đúng như mọi lần:** vòng đọc kết thúc, nó thêm lượt assistant vào history. Lượt này chứa một block tool_use có chuỗi input là JSON bị cắt cụt.
5. **Retry:** người vận hành thấy câu trả lời dở nên retry. Request retry mang theo lượt hỏng trong history, và API từ chối bằng lỗi validation chỉ vào block tool_use sai định dạng.
6. **Debug đi sai hướng:** cả nhóm mất một buổi chiều soi schema và logic retry, vì lỗi hiện ra ở request retry.
7. **Nguyên nhân gốc nằm ở thượng nguồn:** handler coi "vòng đọc kết thúc" tương đương "message hoàn tất". Hai việc này khác nhau.

## Điều cần nhớ (phần "What to watch out for")

- Stream kết thúc ≠ message hoàn tất. Chỉ `message_stop` mới có nghĩa message nguyên vẹn.
- Nếu handler commit lượt vào history mỗi khi vòng đọc thoát, một stream bị ngắt sẽ ghi một block dang dở vào history. Lỗi xuất hiện ở request **sau**, không phải request gây ra nó.

Ba việc phải làm:
1. **Chỉ append vào history sau khi nhận `message_stop`** (điều kiện chặn rõ ràng, không dựa vào việc vòng đọc đã thoát).
2. **Khi stream bị ngắt, bỏ phần lượt dang dở** thay vì lưu lại.
3. **Retry từ lượt hoàn chỉnh gần nhất.**

Mẹo debug: khi gặp lỗi tool-use trên một request retry, **hãy kiểm tra xem lượt trước đó có được lắp từ một stream không** trước khi động vào schema.

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt**

- Agent dùng streaming; handler thêm lượt assistant vào history **khi vòng đọc stream thoát**.
- Test trên mạng local: stream luôn chạy đến `message_stop`, nên không bao giờ gặp lỗi.
- Production: mạng chập chờn làm stream đứt giữa chừng, block tool_use chỉ có nửa JSON input. Handler vẫn commit lượt đó.
- Người vận hành retry, request mang theo lượt hỏng, API từ chối bằng lỗi validation. Lỗi hiện ở **request sau**, nên cả nhóm soi nhầm schema và retry cả buổi chiều.
- Gốc lỗi: coi "vòng đọc kết thúc" là "message hoàn tất". **Chỉ `message_stop` mới xác nhận message trọn vẹn.**
- Cách xử lý: (1) chỉ append sau `message_stop`, (2) stream đứt thì bỏ lượt dở, (3) retry từ lượt hoàn chỉnh gần nhất.
- Mẹo debug: gặp lỗi tool-use ở request retry, kiểm tra lượt trước có được lắp từ stream không trước khi đụng schema.

**ELI5**

Bạn ghi lời nhắn qua điện thoại cho đồng nghiệp: "Chuyển khoản 5 triệu cho...", rồi cuộc gọi rớt. Bạn vẫn dán mẩu giấy dở dang đó lên bảng như một chỉ thị hoàn chỉnh. Hôm sau kế toán đọc không hiểu, và người ta đi kiểm tra... máy in, thay vì cuộc gọi bị rớt hôm qua.

**Ví dụ khi implement**

**Snippet 1: test mô phỏng stream đứt (test thường bỏ sót nhánh này)**

```python
def flaky_stream(events, cut_after):
    for i, e in enumerate(events):
        if i == cut_after:
            return            # kết thúc êm, không có message_stop
        yield e

def test_partial_turn_not_committed():
    messages = []
    with pytest.raises(StreamInterruptedError):
        handle_stream(flaky_stream(EVENTS_WITH_TOOL_USE, cut_after=4), messages)
    assert messages == []     # history không bị nhiễm
```

Dùng để: chứng minh handler không ghi lượt dở vào history.

**Snippet 2: retry từ lượt hoàn chỉnh gần nhất**

```python
for attempt in range(3):
    try:
        return handle_stream(client, list(messages))   # luôn gửi history sạch
    except StreamInterruptedError:
        continue
```
