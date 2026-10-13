---
course: Claude Code, MCP & Integration
series: Claude Certified Developer - Foundations (Prep Course)
module: 3 - Claude Code, MCP & Integration
section: Durable Project Context
screen: 5 of 22
screen_type: TEACHING (bài giảng, khoảng 20 phút)
topic: Durable project context với CLAUDE.md, rules files, hooks và subagents
screen_title: "Durable project context with CLAUDE.md, rules files, hooks, and subagents"
related_screens: Screen 2 (permission modes, settings), Screen 6 (Watch Out - CLAUDE.md phình to), Screen 7 (Checkpoint 2)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần đầu bám đủ nội dung màn gốc, phần "Tóm tắt, ELI5 và ví dụ" ở cuối là phần Claude thêm
---

# [Module 3 · Screen 5 · TEACHING] Durable project context: CLAUDE.md, rules, hooks, subagents

## Mở đầu

Màn trước nói về lớp cấu hình quyết định agent **được phép làm gì** (permission modes, settings). Cụm này xây tiếp lên đó: cấu hình **agent biết gì và cư xử ra sao**, để các quy tắc và ngữ cảnh dự án bạn đặt trong một session **vẫn còn hiệu lực ở đầu session sau**.

## CLAUDE.md: file dự án nạp vào mọi session

- Mỗi lần Claude Code khởi động trong thư mục dự án, nó tìm file `CLAUDE.md` ở thư mục gốc và đọc nó. Nội dung được **nối vào prompt trước khi tin nhắn nào của bạn đến**. Nên mọi quy ước, ràng buộc, lệnh bạn ghi ở đây đều có mặt từ prompt đầu tiên của mọi session, không cần nhắc lại.
- Lệnh **`/init`** quét codebase và sinh một `CLAUDE.md` khởi đầu. Đây là baseline tốt nhưng **phải kiểm lại trước khi dùng**. Hãy tinh chỉnh để nó chứa những quy tắc quyết định kết quả prompt: lệnh test, quy ước framework, đường dẫn agent không được đụng, và các quyết định phong cách khác mặc định.
- **Kích thước là kiểu hỏng chính.** CLAUDE.md cứ lớn dần theo mỗi chỉ dẫn mới sẽ làm loãng những quy tắc quan trọng nhất. File to ăn nhiều context window hơn, khiến mỗi chỉ dẫn chiếm phần nhỏ hơn của những gì được nạp, nên giảm khả năng agent làm theo đúng quy tắc chặn được lỗi thật.
- Cách xử lý: chỉ giữ trong CLAUDE.md những ràng buộc **làm đổi hành vi**; mọi thứ còn lại chuyển sang **Skills** nạp theo yêu cầu.

## Rules files: gắn hướng dẫn vào đúng chỗ cần

- CLAUDE.md luôn bật. **Rules files** thêm một lớp hẹp hơn: hướng dẫn chỉ áp dụng ở một phần codebase thì không nạp vào mọi session.
- Rules files nằm trong thư mục **`.claude/rules/`** và có thể giới hạn theo đường dẫn bằng trường **`paths`** (glob) trong YAML frontmatter. Rule giới hạn như vậy **chỉ nạp vào context khi Claude Code làm việc với file khớp pattern**.
- **Phạm vi đến từ frontmatter, không phải vị trí file.** Có thể chia rules vào thư mục con (ví dụ `.claude/rules/database/`), nhưng đó chỉ là cách tổ chức. Một rules file **không có `paths`** sẽ **nạp vô điều kiện lúc khởi động**, cùng mức ưu tiên với CLAUDE.md, bất kể nằm ở đâu trong `.claude/rules/`.
- Cách chia: bộ nhớ chung và ràng buộc phổ quát → CLAUDE.md; hướng dẫn hẹp theo đường dẫn → rules file có `paths`.
  - "Không bao giờ sửa schema database" → CLAUDE.md (áp dụng khắp nơi).
  - "Mọi SQL trong module database phải có ranh giới transaction rõ ràng" → `.claude/rules/database.md`, frontmatter:

```yaml
---
paths:
  - "src/db/**/*.sql"
---
```

  để rule chỉ vào context khi Claude đang làm việc với các file đó.

## Hooks: chạy script của bạn ở các điểm cố định trong vòng đời

- Hook cho phép **chặn và điều khiển tool call trước hoặc sau khi chúng chạy**.
- Nếu ghi trong CLAUDE.md "chạy Prettier sau mỗi lần sửa file", agent sẽ làm theo **phần lớn thời gian**. Hook thì khiến việc đó xảy ra **mọi lần, không ngoại lệ**, vì hook kích hoạt độc lập với việc model quyết định gì.
- Hook được định nghĩa trong **settings files** và cấu hình bằng lệnh `/hooks`. Mỗi hook gắn với: một **sự kiện vòng đời**, một **matcher tuỳ chọn** (giới hạn theo loại tool), và một **command** chạy khi sự kiện xảy ra.

Các sự kiện cốt lõi:

| Sự kiện | Chạy khi nào | Dùng để |
|---|---|---|
| **PreToolUse** | Trước khi tool call chạy | Kiểm tra tool call và **thoát với exit code 2 để chặn**, ghi lý do vào stderr làm phản hồi cho agent. Đây là cách **ép kiểm soát truy cập ở tầng cấu hình** thay vì hy vọng agent tuân theo CLAUDE.md |
| **PostToolUse** | Sau khi tool call xong | **Không chặn được** (việc đã xảy ra). Hợp cho tác dụng phụ tự động: chạy formatter sau khi sửa, chạy test sau khi đổi file, ghi log audit |
| **UserPromptSubmit** | Khi bạn gửi prompt, trước khi model xử lý | Chèn thêm context hoặc kiểm tra yêu cầu trước khi bắt đầu làm |
| **Stop** | Khi model trả lời xong | Việc cuối lượt: thông báo, dọn dẹp, commit log audit |
| **Notification** | Khi Claude Code gửi thông báo (cần quyền dùng tool, hoặc rảnh 60 giây) | Chuyển tín hiệu đó ra kênh ngoài hoặc hệ thống log |
| **SessionStart** | Khi session bắt đầu hoặc tiếp tục | Khởi tạo trạng thái, kiểm biến môi trường, xác nhận dịch vụ cần thiết truy cập được |
| **SessionEnd** | Khi session kết thúc | Dọn dẹp, ghi audit cuối, thông báo session đã đóng |

Điểm chốt: hook PreToolUse chặn sửa đường dẫn cấu hình production sẽ **ép ràng buộc đó ở mọi tool call, mọi session, bất kể permission mode**. Đó là khác biệt giữa **guardrail** và **quy ước**.

## Subagents: giao việc cho một context riêng biệt

- Subagent là trợ lý chuyên biệt mà Claude Code giao việc cho. Mỗi subagent chạy task trong **context riêng** và **chỉ trả về output**. Nó **không kế thừa** lịch sử hội thoại chính, file bạn đã gom vào context, hay trạng thái session. Nó bắt đầu từ trang trắng, làm xong, trả kết quả.
- Các subagent có sẵn khác nhau ở thứ nạp lúc khởi động, và điều này quyết định quy tắc dự án có áp dụng không. (Trang nhắc kiểm danh sách hiện hành trong tài liệu Claude Code vì tập này đã tăng theo thời gian, nhưng phép chia sau giữ qua các phiên bản.)
  - **Explore** và **Plan** **bỏ qua CLAUDE.md và git status** để nghiên cứu nhanh, rẻ. Nên quy tắc dự án trong CLAUDE.md **không có trong context** của chúng. Nếu giao việc cho Explore/Plan mà có rule CLAUDE.md áp dụng thì rule đó không được nạp.
  - **general-purpose** nạp **cả hai**.
  - Khi ràng buộc dự án bắt buộc phải được tôn trọng: dùng subagent **general-purpose**, hoặc **custom subagent** nạp rõ các rule nó cần.
- **Custom subagent không tự thấy skills.** Nếu định nghĩa subagent trong `.claude/agents` và nó cần skill nào thì phải **liệt kê skill đó trong front matter** của agent. Built-in agent không có skill nạp sẵn; muốn hành vi dựa trên skill thì tạo custom subagent có liệt kê skills.

## Bản đồ so sánh 4 cơ chế (thẻ lật trên trang)

| Cơ chế | Nạp gì | Chạy khi nào | Chi phí context | Hợp để đặt |
|---|---|---|---|---|
| **CLAUDE.md** | Toàn bộ nội dung, nối vào đầu context lúc bắt đầu session | Mọi session, vô điều kiện | Cố định mỗi session, **loãng dần khi file to** | Ràng buộc dự án phổ quát, lệnh, quyết định framework |
| **Rules file** | Nội dung file; giới hạn bằng `paths` trong frontmatter; không có `paths` thì nạp như CLAUDE.md | Khi Claude đọc file khớp `paths`. Rule không giới hạn nạp lúc bắt đầu session | Có `paths`: chỉ tốn khi bị kích hoạt. Không `paths`: tốn cố định như CLAUDE.md | Hướng dẫn theo đường dẫn mà ở chỗ khác chỉ là nhiễu |
| **Hook** | Chạy script ở sự kiện vòng đời; **không thêm nội dung vào context** | Ở sự kiện đã cấu hình (PreToolUse, PostToolUse...) | Tối thiểu: chỉ output của script nếu được chuyển lại cho Claude | Guardrail bắt buộc, tác dụng phụ tự động, log audit |
| **Subagent** | Chỉ context của task, tách biệt khỏi session chính | Khi session chính giao một task | Trả về bản tóm tắt, không phải toàn bộ lịch sử task | Khám phá, điều tra, task mà output sẽ làm phình context chính; việc chia nhỏ song song được |

**Hợp với:** dự án bạn quay lại qua nhiều session, nơi bộ rule ổn định, khác biệt theo thư mục, hoặc guardrail vô điều kiện đáng công cài đặt.
**Dùng cách khác:** việc dùng một lần sẽ không quay lại. Với khám phá nhanh một codebase lạ, chi phí cài đặt không đáng.

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt một câu:** CLAUDE.md là luật chung luôn nạp (giữ ngắn); rules file là luật theo khu vực chỉ nạp khi vào khu vực đó; hook là chốt bảo vệ chạy máy móc mọi lần (PreToolUse chặn được, PostToolUse thì không); subagent là người phụ việc làm trong phòng riêng và chỉ báo kết quả (Explore/Plan không đọc CLAUDE.md).

**ELI5** (mượn hình ảnh toà nhà):
- **CLAUDE.md** = bảng nội quy dán ngay cửa chính, ai vào cũng đọc. Dán quá nhiều thì không ai đọc nổi, quy tắc quan trọng chìm mất.
- **Rules file** = biển nhỏ dán riêng ở cửa phòng thí nghiệm, chỉ ai bước vào phòng đó mới thấy.
- **Hook** = cái cửa từ có bảo vệ máy móc: cứ có người quẹt thẻ là nó chạy, không "quên", không "tuỳ hứng". PreToolUse là cửa từ trước khi vào (chặn được), PostToolUse là camera ghi lại sau khi người ta đã vào (không chặn được nữa).
- **Subagent** = nhờ một bạn vào phòng riêng điều tra rồi ra báo kết quả, bạn không phải xem từng bước. Nhưng bạn "Explore" và "Plan" không đọc bảng nội quy cửa chính, nên nội quy không áp dụng cho họ.

**Quy tắc chọn nhanh (cho đề thi)**
- "Phải xảy ra **mọi lần**, không được quên" → **Hook** (chặn trước = PreToolUse).
- "Áp dụng **khắp dự án**" → **CLAUDE.md** (ngắn gọn!).
- "Chỉ áp dụng **ở thư mục/loại file X**" → **Rules file + `paths`**.
- "Việc khám phá/điều tra **sẽ làm phình context**" → **Subagent**.
- "Cần quy tắc CLAUDE.md được tôn trọng" → đừng dùng Explore/Plan, dùng general-purpose hoặc custom.

**Snippet 1: hook PreToolUse chặn sửa đường dẫn production** (ví dụ do Claude viết, trang không in code này; cấu trúc cụ thể cần đối chiếu tài liệu hooks)

```bash
#!/usr/bin/env bash
# PreToolUse hook: đọc tool call (JSON) từ stdin, chặn nếu đụng /deploy/config/prod/
input="$(cat)"
if echo "$input" | grep -q "/deploy/config/prod/"; then
  echo "Bị chặn: không được sửa cấu hình production." >&2   # lý do agent sẽ thấy
  exit 2                                                      # exit 2 = chặn tool call
fi
exit 0
```

Dùng để: minh hoạ ý "exit code 2 + lý do ở stderr" của PreToolUse. Ràng buộc này áp dụng ở mọi tool call, mọi session, bất kể permission mode.

**Snippet 2: rules file có `paths`**

```markdown
---
paths:
  - "src/db/**/*.sql"
---
Mọi câu SQL trong module database phải có ranh giới transaction rõ ràng.
```

Dùng để: file `.claude/rules/database.md` này chỉ nạp khi Claude làm việc với file SQL khớp pattern. Nếu bỏ phần `paths`, nó thành rule nạp vô điều kiện giống CLAUDE.md.
