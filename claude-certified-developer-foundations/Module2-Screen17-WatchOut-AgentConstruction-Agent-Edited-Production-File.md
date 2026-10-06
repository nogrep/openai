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
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học
---

# [Module 2 · Screen 17 · WATCH OUT] Agent đã sửa một file production

## Tóm tắt ngắn

- Agent có 3 tool: `read_file`, `write_file`, `validate_config`. Vòng lặp: ghi xong thì validate, lỗi thì sửa tiếp, tối đa 10 vòng.
- Test trong thư mục tạm: chạy tốt, 2-3 vòng là xong.
- Ở môi trường khách hàng: agent sửa một **rate limit** mà ứng dụng khách hàng đang dựa vào, `validate_config` báo pass (giá trị nằm trong khoảng schema cho phép), vòng lặp thoát sau 1 vòng. Vài phút sau ứng dụng khách hàng lỗi vì bị throttle.
- **Lỗi không nằm ở vòng lặp.** Điều kiện thoát chỉ kiểm "file này hợp lệ", không kiểm "ghi vào hệ thống thật có an toàn không". Thiếu một **checkpoint con người (HITL)** trước lần ghi đầu tiên vào config thật.
- Câu hỏi thiết kế cần đặt từ đầu: **"Nếu `write_file` chạy mà không ai kiểm, kết quả xấu nhất là gì?"** Tool nào làm được hành động không hoàn tác ở production thì phải có checkpoint trước khi chạy, và quyết định điều này **lúc thiết kế**, không đợi sau sự cố.

## ELI5

Bạn nhờ cậu bé giúp việc chỉnh nút âm lượng cái loa. Lúc tập ở nhà, cậu chỉnh sai thì chẳng sao. Hôm nay đến nhà hàng, cậu thấy nút "hơi quá tay" nên chỉnh nhỏ xuống, đúng luật của cái loa (nằm trong vạch cho phép), rồi báo "xong!". Nhưng nhà hàng đang chạy nhạc nền cho cả buổi tiệc, nhỏ quá thì khách không nghe được gì.

Cậu bé không làm sai việc được giao. Cái thiếu là một câu hỏi trước khi vặn nút: **"Chú ơi, con chỉnh thế này được không?"** Việc "giá trị hợp lệ" (đúng vạch) khác hẳn việc "an toàn cho buổi tiệc đang chạy".

## Ví dụ thực tế khi implement

Ý tưởng: ở production, tool ghi không ghi ngay mà đưa diff cho người duyệt trước.

**Snippet 1: chốt chặn trong tool ghi**

```python
if env in PROTECTED_ENVS:
    decision = approver(path=path, diff=make_diff(old, new))
    if decision != "approve":
        return {"status": "rejected_by_human"}
open(path, "w").write(new)
```

Dùng để: chỉ môi trường production mới phải chờ người duyệt, test vẫn ghi thẳng. Chốt chặn nằm trước dòng ghi, không nằm ở `validate_config`.

**Snippet 2: điều kiện thoát của vòng lặp**

```python
for _ in range(MAX_ITERATIONS):
    resp = client.messages.create(...)
    if resp.stop_reason != "tool_use":
        break
```

Dùng để: vòng lặp dừng khi Claude không gọi tool nữa, có giới hạn số vòng. Khi người từ chối, agent nhận `rejected_by_human` làm tool_result và đổi cách làm, vòng lặp không vỡ.
