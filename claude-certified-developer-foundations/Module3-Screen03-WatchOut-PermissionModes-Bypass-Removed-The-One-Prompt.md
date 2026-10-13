---
course: Claude Code, MCP & Integration
series: Claude Certified Developer - Foundations (Prep Course)
module: 3 - Claude Code, MCP & Integration
section: Permission Modes và Human Gates
screen: 3 of 22
screen_type: WATCH OUT (lỗi thường gặp, khoảng 3 phút)
topic: Permission Modes và Human Gates
screen_title: "The bypass mode that removed the one prompt that mattered"
related_screens: Screen 2 (Teaching - permission modes, settings, human gate), Screen 4 (Checkpoint 1 - lắp settings file và đặt human gate)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần đầu bám đủ nội dung màn gốc, phần "Tóm tắt, ELI5 và ví dụ" ở cuối là phần Claude thêm
---

# [Module 3 · Screen 3 · WATCH OUT] Bypass mode đã tắt mất đúng cái prompt quan trọng nhất

## Bối cảnh (phần "Setup")

Bạn chuyển sang `bypassPermissions` để hết bị hỏi liên tục, vì công việc có vẻ thường lệ. Agent đã chạy ngoan suốt nhiều ngày, task dọn dẹp thì đơn giản, và việc bị hỏi trước mỗi tool call nghe như ma sát thừa.

## Case study: buổi pairing dọn code chạy êm suốt ba ngày

Đợt dọn dẹp codebase đã chạy ba ngày không sự cố. Sau đó:

- **Dev:** "Mình chuyển sang `bypassPermissions` cho lần này. Chỉ là đổi tên các tham chiếu endpoint cũ, không có gì rủi ro."
- **Claude Code:** quét file khớp pattern `/v1/legacy/` và thấy **47 file**.
- **Claude Code:** cập nhật các tham chiếu endpoint, rồi chạy script `cleanup.sh` (bước dọn sau khi đổi tên).
- **Claude Code:** xoá **3 file** khớp `/v1/legacy/` nằm trong `/deploy/config/prod/`.
- **Dev:** "Khoan. Trong `/deploy/config/prod/` có gì?"
- **Claude Code:** đó là các file override endpoint theo môi trường của deployment production, giờ đã bị xoá.
- **Dev:** "Thư mục đó đâu có nằm trong phạm vi. Mình đang làm ở `/src/`."
- **Claude Code:** pattern `/v1/legacy/` khớp file ở cả `/src/` lẫn `/deploy/config/prod/`. Ở `bypassPermissions` không có xác nhận nào được hỏi trước khi xoá.

## Vì sao không ai chặn được

Cái prompt có thể bắt được lỗi này đã bị tắt cùng lúc với việc bật `bypassPermissions`. Ở `default` hoặc `acceptEdits`, script dọn dẹp sẽ **không chạy nếu chưa xác nhận**, và dev có thể dừng việc xoá trước khi nó chạm vào config production. Ở `bypassPermissions`, pattern khớp rộng hơn dự định và **không có prompt nào đứng giữa** script và các file bị xoá.

### Chi tiết vị trí của "cổng" (gate)

Cái cổng ở đây là **lệnh gọi script**, không phải bản thân các lệnh `rm`:

- `acceptEdits` tự duyệt các lệnh filesystem thông dụng, **kể cả `rm` trên đường dẫn trong working directory**.
- Nếu Claude xoá bằng lệnh `rm` trực tiếp, `acceptEdits` sẽ cho qua im lặng.
- Chỉ `default` mới hỏi trước các lệnh `rm` đó.

## Điều cần nhớ (phần "What to Watch Out for")

1. **Bypass mode tắt mọi prompt xác nhận, kể cả những prompt bạn không nghĩ tới.** Kiểu lỗi: agent khớp một tập file **rộng hơn bạn định**, trong một mode **không có checkpoint**.
2. **`bypassPermissions` còn bỏ luôn "protected path guard"** mà các mode khác vẫn giữ. Nghĩa là ngay cả trạng thái repo và cấu hình của chính Claude cũng mất prompt tự động.
3. **Cách bù:** đặt **deny rule** cho các thư mục nhạy cảm **trước khi** chuyển mode.
4. **Muốn ít prompt mà vẫn giữ lưới an toàn:** dùng mode có **classifier chặn cổng** (ví dụ `auto`) thay vì bypass hoàn toàn.

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt một câu:** bypass không chỉ bỏ những prompt phiền, nó bỏ luôn prompt duy nhất sẽ cứu bạn; nếu muốn ít bị hỏi thì đặt deny rule trước, hoặc dùng `auto` thay vì bypass.

**ELI5:** bạn tắt hết chuông báo cháy trong nhà vì nó hay kêu oan. Mười ngày không sao. Hôm thứ mười một có cháy thật thì không cái chuông nào kêu. Lỗi không nằm ở việc "hôm đó đặc biệt nguy hiểm", mà ở chỗ bạn không còn cách nào biết trước.

**So sánh nhanh theo đúng nội dung màn này**

| Mode | Hỏi trước khi chạy script `cleanup.sh`? | Hỏi trước `rm` trực tiếp? | Protected path guard |
|---|---|---|---|
| `default` | Có | Có | Có |
| `acceptEdits` | Có | Không (tự duyệt `rm` trong working directory) | Có |
| `bypassPermissions` | Không | Không | Không |

(Bảng do Claude tổng hợp từ các chi tiết của trang; trang không in bảng này.)

**Snippet minh hoạ: deny rule đặt trước khi bypass**

```json
{
  "permissions": {
    "deny": [
      "Edit(/deploy/config/prod/**)",
      "Bash(rm:/deploy/config/prod/**)"
    ]
  }
}
```

Dùng để: chặn cứng thư mục production cho dù mode có lỏng đến đâu. Đây là ví dụ do Claude viết theo mô tả "đặt deny rule cho thư mục nhạy cảm"; trang này không in code, và cú pháp rule cụ thể cần đối chiếu với tài liệu settings của Claude Code. Màn 4 (Checkpoint 1) là nơi thực hành lắp settings file.

**Mẹo nhớ cho đề thi:** câu hỏi kiểu này thường hỏi "prompt nào sẽ bắt được lỗi, ở mode nào". Nhớ hai ý: (1) cổng là **lệnh gọi script**, không phải `rm`; (2) `acceptEdits` vẫn cho `rm` trong working directory đi qua, chỉ `default` hỏi.
