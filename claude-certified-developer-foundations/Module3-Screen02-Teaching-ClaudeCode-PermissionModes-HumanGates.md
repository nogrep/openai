---
course: Claude Code, MCP & Integration
series: Claude Certified Developer - Foundations (Prep Course)
module: 3 - Claude Code, MCP & Integration
section: Permission Modes và Human Gates
screen: 2 of 22
screen_type: TEACHING (bài giảng, khoảng 17 phút)
topic: Claude Code agent loop, permission modes, settings, và chỗ đặt human gate
screen_title: "Permission Modes · Human Gates"
related_screens: Module 2 (agent loop ở mức API, HITL checkpoint)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần cuối "Tóm tắt, ELI5 và ví dụ" là phần Claude thêm
---

# [Module 3 · Screen 2 · TEACHING] Permission Modes và Human Gates

## Mở đầu

Module 2 đã nói về agent loop ở mức API: model gọi tool, nhận kết quả, rồi tiếp tục cho đến khi xong việc. Claude Code chạy **cùng vòng lặp đó** trong terminal của bạn, nhưng thêm một lớp nữa: **hệ thống permission chặn mọi hành động** agent muốn làm. Trước khi cấu hình được gì, bạn cần hiểu vòng lặp chạy thế nào và các permission mode điều khiển điều gì.

## Claude Code xử lý một task thế nào: explore, plan, code

Khi bạn giao task, Claude Code **không viết code ngay**:
1. **Explore (khám phá):** đọc file, lần theo logic liên quan, dựng bức tranh về codebase.
2. **Plan (lập kế hoạch):** khi đã hiểu đủ để đề xuất thay đổi, nó tạo một plan, tức là mô tả có cấu trúc các chỉnh sửa nó định làm.
3. **Code (viết code):** chỉ sau khi bạn xem và duyệt plan, nó mới bắt đầu viết và chạy thay đổi.

Trình tự này quan trọng vì hai lý do:
- **Kết quả tốt hơn:** hiểu codebase trước khi đụng vào nên ít phải đoán và bắt được nhiều tác động dây chuyền hơn.
- **Đây là chỗ permission mode cắm vào:** **plan mode** giữ Claude Code ở pha explore, chặn mọi sửa file và lệnh shell cho đến khi bạn thả ra. Nên nó là mặc định hữu ích cho codebase lạ hoặc việc rủi ro cao.

## Permission modes: duyệt, cổng chặn và ràng buộc

Permission mode điều khiển **Claude Code dừng lại hỏi xác nhận thường xuyên đến đâu**. Mỗi mode đánh đổi khác nhau giữa tốc độ và mức giám sát. Chọn mode nào tùy bạn hiểu codebase đến đâu và thay đổi dễ hoàn tác đến đâu.

Trang web có sáu tab, mỗi tab nói: tự duyệt gì, vẫn chặn gì, giới hạn gì.

| Mode | Tự duyệt | Vẫn chặn (cần xác nhận) | Giới hạn / dùng khi nào |
|---|---|---|---|
| **default** | Chỉ việc đọc | Mọi sửa file và lệnh shell | An toàn nhưng chậm trên việc đã tin cậy. Là mức nền cho dự án mới hoặc codebase lạ |
| **acceptEdits** | Đọc, sửa file, và các lệnh filesystem thông dụng (`mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, `sed`) **trong thư mục làm việc**. Auto-approve chỉ giới hạn trong đường dẫn thuộc thư mục làm việc, đường dẫn được bảo vệ (protected path) vẫn hỏi | Mọi lệnh shell khác; ghi ra ngoài thư mục làm việc; ghi vào protected path | Việc local đã tin cậy mà chạy shell vẫn cần người nhìn. Không hợp khi agent phải chạy script |
| **plan** | Chỉ đọc. Nghiên cứu và đề xuất, không sửa gì | Mọi sửa file và lệnh shell cho đến khi bạn duyệt plan | Khám phá và lập kế hoạch trên codebase nhạy cảm hoặc lạ. Không hợp với task bắt buộc phải ghi output |
| **auto** | Mọi thứ, nhưng một **classifier riêng** xem xét từng hành động trước và chặn thứ vượt quá yêu cầu của bạn, nhắm vào hạ tầng không nhận ra, hoặc có vẻ do nội dung thù địch hay không phù hợp thúc đẩy | Mặc định chặn: deploy production và migration, xóa hàng loạt, đánh cắp credential, force-push lên main | Giảm số lần hỏi nhưng **không đảm bảo an toàn**: đây là research preview, không thay thế việc tự xem các thao tác nhạy cảm. Khả dụng tùy gói, phiên bản model và cài đặt admin. Luôn kiểm tra yêu cầu hiện hành trước khi xây dựng dựa vào nó |
| **dontAsk** | Chỉ các tool bạn đã duyệt trước bằng **allow rule**, cộng lệnh chỉ đọc. **Tự động TỪ CHỐI** mọi thứ còn lại | Mọi tool call không nằm trong allow list đều bị từ chối. Không có hàng chờ xác nhận | Dành cho CI và script khóa chặt. Hạn chế tốt nhưng không phải cách giảm ma sát cho việc tương tác local |
| **bypassPermissions** | Mọi tool call. Không hỏi, không kiểm tra an toàn | Bình thường là không gì cả: các kiểm tra permission chuẩn bị bỏ qua. Chỉ các lệnh xóa thảm họa như `rm -rf /` và `rm -rf ~` vẫn kích hoạt một prompt phút chót | Chỉ trong container hoặc VM cô lập, dùng xong bỏ. **Không bao giờ** dùng trên máy dev với codebase thật |

## Cấu hình nằm ở đâu và áp dụng cho ai

Settings đặt ở nhiều tầng, mỗi tầng quyết định phạm vi của các rule trong đó:

- **User level** (`~/.claude/settings.json`): áp dụng cho mọi dự án trên máy. Chỗ hợp cho sở thích đi theo bạn khắp nơi, ví dụ mode mặc định cho việc khám phá.
- **Project level** (`.claude/settings.json`, commit vào repo): áp dụng cho mọi người clone repo. Chỗ hợp cho quy ước cả team, allow rule cho các tool dự án dùng, và deny rule cho các đường dẫn không được đụng.
- **Local project level** (`.claude/settings.local.json`): ghi đè cá nhân cho một dự án, tự động bị git-ignore. Chỗ hợp cho sở thích riêng không nên commit cho cả team.
- **Enterprise level** (`managed-settings.json`, do admin đặt): **user và file dự án không ghi đè được**. Chỗ hợp cho kiểm soát an ninh toàn tổ chức, ví dụ cấm sửa file môi trường (env) hoặc chặn một số lệnh shell trên mọi dự án.

Allow rule và deny rule **xếp chồng lên mode đang chọn**. **Deny rule luôn thắng allow rule, bất kể mode nào đang hiệu lực.** Kiểm soát quản trị bền nhất là deny rule ở tầng enterprise: không dev nào gỡ được, và nó áp dụng cả khi đang đặt chế độ bypass.

## Chỗ con người vẫn phải nhìn: đặt review gate theo chi phí xấu nhất

Permission mode và deny rule quyết định agent **làm được gì mà không hỏi**. Chúng không quyết định **chỗ con người vẫn phải nhìn** trước khi một hành động có hiệu lực. Quyết định đó dựa vào một câu hỏi, cũng chính là câu hỏi phân biệt mode an toàn với mode rủi ro:

> **Kết quả tệ nhất là gì nếu hành động này chạy mà không ai kiểm tra?**

Chi phí khi sai càng thấp thì càng cho qua được nhiều. Chi phí càng cao và càng khó hoàn tác thì bước đó càng cần human gate trước khi thực thi.

Cùng câu hỏi "tệ nhất là gì" đặt gate đúng chỗ dù agent đang viết code hay chạy không người trông trong một bước tự động (ví dụ bot comment hoặc chặn pull request). Từ đó có ba cách đặt:

1. **Cho qua, không gate, với hành động rủi ro thấp và hoàn tác được.** Sửa format, hay một chỉnh sửa nằm trong thư mục làm việc, sai cũng ít tốn. Bắt người duyệt từng cái là mua sự giám sát bạn không cần và làm chậm việc. Đây chính là trường hợp `acceptEdits` sinh ra để phục vụ.
2. **Gate mọi hành động khó hoàn tác hoặc chạm đường dẫn nhạy cảm:** ghi ra ngoài thư mục làm việc, lệnh shell phá hủy, sửa file liên quan bảo mật hay file được bảo vệ. Sai ở đây tốn rất nhiều nên agent phải dừng và đưa hành động ra cho người trước khi chạy. **Deny rule ép điều này một cách tất định** (deterministic), còn mode `default` hoặc `plan` giữ nguyên prompt để bạn quyết định.
3. **Không bao giờ để agent là cổng duy nhất** cho thay đổi vào code mà team đã đánh dấu là nhạy cảm. Ở đó công việc của agent chỉ là **đầu vào cho quyết định của con người**, không thay thế quyết định đó: phải có người review trước khi merge, dù agent hay phần tự review của nó có tự tin đến đâu.

Chọn permission mode và đặt gate là **cùng một quyết định nhìn từ hai phía**: mode đặt mặc định cho cả session, còn gate là chỗ bạn ghi đè mặc định đó cho đúng hành động có chi phí quá cao để phó mặc. Cả hai đều xuất phát từ câu hỏi "cái gì hỏng nếu việc này chạy không kiểm tra?".

## Cost, Complexity, Risk

- **Cost:** chạy `default` trên việc đã tin cậy thêm độ trễ do prompt vào mỗi tool call, và độ trễ này cộng dồn trong một đợt refactor dài.
- **Complexity:** nhiều tầng settings và file với thứ tự ghi đè đòi hỏi cẩn thận nhất quán. Một deny rule ở tầng enterprise mâu thuẫn với allow rule ở tầng project thì mọi người bảo trì dự án phải hiểu rõ.
- **Risk:** rủi ro nằm ở dùng sai mode cho ngữ cảnh. Ví dụ bật chế độ bypass vì sốt ruột trên máy không cô lập sẽ gỡ mọi prompt an toàn giữa agent và file thật của bạn, và khác với các mode khác, nó còn **gỡ luôn lớp bảo vệ protected-path**.

---

## Tóm tắt, ELI5 và ví dụ

**Tóm tắt**

- Claude Code = cùng agent loop như Module 2 + lớp permission chặn mọi hành động. Nó làm theo 3 pha: **explore, plan, code**. Plan mode giữ nó ở explore.
- 6 mode, từ chặt đến lỏng: `plan` (chỉ đọc), `default` (đọc, còn lại hỏi), `acceptEdits` (thêm sửa file trong thư mục làm việc), `auto` (classifier soi từng việc, research preview), `dontAsk` (chỉ chạy allow list, còn lại từ chối, dành cho CI), `bypassPermissions` (không hỏi gì, chỉ trong VM/container dùng xong bỏ).
- 4 tầng settings: user, project (commit), local (git-ignore), enterprise (không ai ghi đè). **Deny luôn thắng allow.**
- Đặt gate bằng câu hỏi "tệ nhất là gì nếu không ai kiểm tra?": rẻ và hoàn tác được thì cho qua, đắt hoặc khó hoàn tác thì gate, code nhạy cảm thì luôn cần người duyệt trước khi merge.
- Cost: prompt làm chậm. Complexity: nhiều tầng rule. Risk: sai mode, nặng nhất là bypass trên máy thật.

**ELI5**

Hình dung Claude Code là một thợ sửa nhà bạn thuê. Pha explore là thợ đi quanh xem nhà trước, pha plan là thợ đưa bạn bản báo giá và kế hoạch, pha code là thợ bắt tay vào làm sau khi bạn gật đầu. Permission mode là mức bạn tin thợ: `plan` là "chỉ được nhìn và báo giá, đừng đụng gì"; `default` là "cái gì động tay cũng phải hỏi tôi"; `acceptEdits` là "trong phòng khách cứ tự sắp xếp đồ đạc, nhưng đụng tường hay điện thì hỏi"; `auto` là có một giám sát viên đứng cạnh xem thợ làm gì có quá đáng không; `dontAsk` là đưa thợ một danh sách việc được làm, ngoài danh sách thì thợ không làm, không hỏi lại; `bypassPermissions` là đưa chìa khóa và đi vắng, chỉ nên làm với căn nhà thử đằng nào cũng đập đi. Deny rule là dán tờ "tuyệt đối không động vào phòng két" lên cửa, thợ cãi cũng vô ích, và tờ của chủ tòa nhà (enterprise) thì người thuê không gỡ được. Còn human gate là lúc thợ phải gọi bạn: việc gì nhỡ làm hỏng thì sửa rất tốn kém hoặc không sửa được thì phải gọi.

**Ví dụ khi implement**

*Snippet 1: `.claude/settings.json` của project, allow cho tool dự án dùng, deny cho file không được đụng*

```json
{
  "permissions": {
    "allow": ["Bash(npm test)", "Bash(npm run lint)"],
    "deny": ["Read(./.env)", "Edit(./.env)", "Bash(rm -rf *)"]
  }
}
```

Dùng để: cả team chia sẻ cùng ranh giới. Deny thắng allow, nên dù ai đó đặt mode lỏng, file `.env` vẫn không bị sửa. (Cú pháp rule chỉ minh họa ý tưởng, kiểm tra lại tài liệu Claude Code hiện hành khi dùng thật.)

*Snippet 2: mode theo ngữ cảnh*

```text
Khám phá codebase lạ          -> plan
Refactor trong repo quen      -> acceptEdits
CI chạy script khóa chặt      -> dontAsk + allow list
Sandbox VM dùng xong bỏ       -> bypassPermissions
Máy dev, codebase thật        -> KHÔNG dùng bypassPermissions
```

Dùng để: chọn mode theo độ quen codebase và độ hoàn tác của thay đổi, đúng tinh thần câu hỏi "tệ nhất là gì".
