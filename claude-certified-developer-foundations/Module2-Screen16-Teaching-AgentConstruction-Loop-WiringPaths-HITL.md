---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Agent construction
screen: 16 of 29
screen_type: TEACHING (bài giảng, khoảng 22 phút)
topic: Agent construction
screen_title: Building a production agent - the loop, wiring paths, orchestration, and human-in-the-loop
related_screens: Screen 13-15 (Context engineering - agent chạy nhiều lượt làm context phình), Screen 10-12 (Streaming - lưu ý commit lượt assistant), Module 4 (bảo mật, IAM, guardrails - bài này chỉ trỏ tới)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Các chi tiết phụ thuộc phiên bản (beta header, ZDR/HIPAA, trạng thái FedRAMP) là điều bài dặn phải kiểm tra lại với tài liệu chính thức
---

# [Module 2 · Screen 16 · TEACHING] Xây agent cho production

## Một câu tóm tắt

Trước khi viết dòng code đầu tiên, hãy quyết định (1) có thật sự cần agent không, (2) chạy vòng lặp bằng đường nào (tự viết / Agent SDK / Managed Agents), và (3) ràng buộc dữ liệu nào quyết định endpoint và credentials. Vòng lặp thì giống nhau ở cả ba đường.

## 1. Workflow hay agent: quyết định trước khi viết code

Bài nhấn mạnh sai lầm lớn nhất là chọn nhầm mẫu hình ngay từ đầu. Dùng agent khi workflow là đủ thì thêm độ phức tạp hành vi mà không thêm năng lực. Dùng workflow khi cần agent thì hệ thống gãy mỗi khi đầu vào lệch khỏi đường đã định.

| Chọn workflow khi | Chọn agent khi |
|---|---|
| Liệt kê được các bước chính xác trong code | Chỉ định được mục tiêu và tool nhưng không chỉ định được đường đi |
| Chi phí lỗi cao, cần guardrail từng bước | Đường đi không thể liệt kê trước |
| Cần quan sát bằng công cụ vận hành thông thường | Chấp nhận tính không tất định, hành động của agent bị giới hạn bởi bộ tool đã đăng ký |
| Đầu vào được ràng buộc trong một tập đã biết | Đầu vào người dùng biến thiên khó đoán về nội dung và cấu trúc |
| Mỗi lần chạy theo cùng một trình tự | Nhiệm vụ cần sắp xếp các tool một cách sáng tạo |

Các ý đi kèm:
- Agent mang chi phí: hành vi phức tạp hơn (đường đi nảy ra từ suy luận trên context tích lũy thay vì rẽ nhánh trong code), và quan sát cần **công cụ ở mức transcript** chứ log vận hành thông thường không đủ.
- Cầu thang độ phức tạp: **một lời gọi API, rồi workflow, rồi agent**. Chỉ nâng bậc khi bậc thấp hơn không xử lý nổi mức biến thiên của bài toán.
- Khi chạy nhiều lượt, các lỗi mà test đơn lượt không thấy sẽ lộ ra: context đầy nhanh hơn dự kiến, bước sau nhận sai đầu vào vì tool call trước cấu trúc sai (nối với Screen 14-15).

> Ví von của Claude: workflow là tàu chạy trên ray, agent là xe có tài xế. Ray đã có thì đừng thuê tài xế; chỉ khi đường chưa biết trước mới cần tài xế, và khi đó bạn phải chấp nhận khó dự đoán hơn và cần camera hành trình (transcript tooling).

## 2. Agent là mẫu hình, đường nối dây (wiring) là lựa chọn triển khai

- Quyết định cần agent đồng nghĩa chọn một mẫu hình: một vòng lặp gọi tool, quản lý context, chạy đến khi đạt mục tiêu. Với agent đơn, mẫu hình này **giống nhau ở cả ba đường**.
- Hệ nhiều agent (planner, executor, evaluator tách riêng, giao việc qua artifact có cấu trúc) thêm các quyết định thiết kế ngoài vòng lặp, và bài hẹn sẽ học ở phần sau của khóa.
- Ba đường nằm trên một phổ **bạn sở hữu bao nhiêu hạ tầng**. Chọn theo ràng buộc triển khai và tuân thủ, không chọn đường nào nhanh nhất để làm prototype.

## 3. Ba đường nối dây: ai chạy vòng lặp, bạn gánh gì

| | Raw Messages API loop | Agent SDK | Claude Managed Agents |
|---|---|---|---|
| Ai chạy vòng lặp | Code của bạn: gửi request, đọc tool-use block, chạy tool, nối kết quả | SDK chạy vòng lặp trong process của bạn; code bạn vẫn chạy tool | Anthropic chạy vòng lặp và sandbox; app gửi event vào và nhận kết quả qua server-sent events |
| Bạn sở hữu | Toàn bộ: vòng lặp, chạy tool, quản lý context, retry, điều kiện thoát | Việc chạy tool và ứng dụng bao quanh; SDK lo cấu trúc vòng lặp, context, đăng ký tool | Lớp ứng dụng và định nghĩa agent (model, system prompt, tool, MCP server, skill) khai báo một lần rồi gọi theo ID |
| Chọn khi | Cần kiểm soát từng bước, ràng buộc mà thư viện không đáp ứng, hoặc đang học cách vòng lặp hoạt động | Muốn dùng đúng khung vòng lặp, xử lý context và scaffold tool đứng sau Claude Code, chạy trong môi trường của mình bằng Python hoặc TypeScript | Tác vụ chạy dài, muốn sandbox được quản lý, không muốn tự dựng vòng lặp, sandbox và lớp chạy tool |
| Cần kiểm tra trước khi chốt | Chi phí bảo trì thuộc về bạn: mọi thứ SDK cho miễn phí (quản lý context, xử lý tool song song) thành code bạn phải viết và test | Việc nạp tính năng dựa trên filesystem (CLAUDE.md, skill) do cấu hình `settingSources` quyết định. **Đừng dựa vào mặc định**: đặt tường minh, ví dụ `["user","project","local"]` để giống Claude Code CLI, hoặc `[]` để chạy cô lập hoàn toàn. Đối chiếu mặc định hiện hành với tài liệu tham chiếu Agent SDK | Đang ở public beta (bề mặt có thể đổi giữa các bản phát hành), cần header beta; xem ràng buộc ZDR/HIPAA bên dưới |

(Tên header beta tôi đọc lúc trước là `managed-agents-2026-04-01`. Đây là chi tiết dễ đổi, hãy kiểm tra tài liệu.)

### Managed Agents: khi nào dùng

Cái bạn thôi sở hữu: vòng lặp lặp, sandbox thực thi, retry bên trong vòng lặp và runtime chạy tool (Anthropic chạy hết phía server). Cái bạn gánh thêm: định nghĩa agent như một tài nguyên API có phiên bản, cộng với lớp ứng dụng gửi event và tiêu thụ kết quả stream. Hai dòng nữa trong bảng bài:
- **Thời lượng và trạng thái phiên:** phiên có thể chạy hàng phút đến hàng giờ mà process của bạn không phải giữ vòng lặp mở. Phiên có trạng thái, **lưu phía Anthropic**, chịu chính sách xử lý dữ liệu của họ.
- **Vòng đời sandbox:** Anthropic lo dựng và hủy; bạn phụ thuộc vào bộ tool và mô hình thực thi của sandbox được quản lý thay vì môi trường của mình.

Chọn Managed Agents khi:
1. Tác vụ chạy dài (hàng phút, hàng giờ), khó giữ mở trong process của bạn.
2. Bạn cần sandbox được quản lý, nếu không thì phải tự dựng và tự bảo mật môi trường thực thi.
3. Không muốn tự xây vòng lặp, sandbox, lớp chạy tool, và chấp nhận định nghĩa agent như một tài nguyên API.

**Ràng buộc quyết định với dữ liệu quy định (hộp cảnh báo đỏ trong bài):** phiên Managed Agents có trạng thái và lưu phía server, nên hiện **chưa đủ điều kiện cho Zero Data Retention (ZDR) hay HIPAA BAA**. Nếu khối lượng công việc có PHI hoặc yêu cầu ZDR thì đường này bị loại, bất kể vận hành tiện đến đâu; hãy chuyển sang Agent SDK hoặc raw loop trên cấu hình được bao phủ. Nguyên tắc bài rút ra: **ràng buộc chi phối chọn đường trước, tiện lợi không có tiếng nói**.

Tiến trình thường gặp: làm prototype với Agent SDK ở local, rồi chuyển sang Managed Agents cho production. Định nghĩa agent cốt lõi mang sang được về mặt khái niệm, nhưng **định dạng đổi** (Agent SDK dùng cấu hình mức code và filesystem, Managed Agents định nghĩa agent như tài nguyên API có phiên bản), nên cần bước **diễn đạt lại**, không phải export trực tiếp.

Thẻ tóm tắt cuối mục:
- **Xử lý tốt:** agent chạy dài, và khối lượng mà bạn không muốn tự dựng sandbox và vòng lặp.
- **Thêm chi phí hoặc phức tạp:** phiên có trạng thái phía server, định dạng agent-như-tài-nguyên, bề mặt beta có thể đổi.
- **Dùng cách khác khi:** PHI/ZDR hoặc cần kiểm soát hoàn toàn trong process, thì ở lại Agent SDK hoặc raw loop trên cấu hình được bao phủ.

## 4. Nối vòng lặp: bốn bước đúng ở mọi đường

Với raw API bạn tự viết cả bốn. Với Agent SDK, SDK cho khung đăng ký tool, đặt system prompt và lặp vòng, còn code bạn vẫn xử lý việc chạy tool. Bước giống nhau, khác ở lượng bạn viết và lượng bạn thừa hưởng.

1. **Đăng ký tool:** mỗi tool theo cùng cấu trúc schema; SDK đăng ký chúng vào agent để Claude biết có gì.
2. **Đặt system prompt:** giới hạn theo nhiệm vụ của agent. Prompt rộng thì định tuyến tool rộng và kém tin cậy; prompt gọi tên nhiệm vụ cụ thể và các tool có sẵn thì hành vi nhất quán hơn.
3. **Xử lý vòng tool-use:** dù bạn tự lặp hay SDK lặp, **code của bạn thực thi** mọi tool call Claude phát ra và trả về trong một khối tool_result.
4. **Định nghĩa điều kiện thoát:** vòng lặp chạy đến khi nhận được điều kiện dừng. Không có điều kiện thoát tường minh, agent tiếp tục gọi tool vượt quá yêu cầu của nhiệm vụ. Phải định nghĩa "xong là xong".

### Checklist nối vòng lặp (kiểm tra bất kể đường nào)

| # | Mục | Cần kiểm tra |
|---|-----|--------------|
| 1 | Tool đã đăng ký | Mọi tool agent có thể cần đều nằm trong danh sách đăng ký; không tham chiếu tool chưa đăng ký trong system prompt |
| 2 | System prompt được giới hạn | Prompt gọi tên nhiệm vụ và tool khả dụng; không mô tả tool agent không có; không bỏ sót tool agent có mà cần hướng dẫn phạm vi |
| 3 | Vòng tool-use đã cài | Code xử lý mọi tool-use block Claude phát ra và trả một tool_result cho từng cái trước lượt assistant kế tiếp; **mọi tool-use block trong cùng một lượt assistant phải được giải quyết cùng nhau** |
| 4 | Đã xác định điểm HITL | Ít nhất một điểm trong vòng lặp có kiểm tra con người |
| 5 | Đã định nghĩa điều kiện thoát | Tiêu chí dừng rõ ràng, **không dựa vào việc Claude tự nguyện dừng** |

## 5. Human-in-the-loop (HITL): chèn ở đâu

Checkpoint HITL tạm dừng agent và chuyển sang bước người duyệt trước khi đi tiếp. **Câu hỏi quyết định vị trí chèn:** kết quả xấu nhất là gì nếu bước này chạy mà không có người kiểm?

| Điểm chèn | Điều kích hoạt kiểm tra | Mức rủi ro xử lý |
|---|---|---|
| Trước một tool call phá hủy | Agent sắp thực hiện thao tác ghi, xóa hoặc gửi | **Cao:** hành động không hoàn tác được, gọi sai thì không sửa được |
| Sau bước lập kế hoạch | Agent vừa sinh kế hoạch và sắp bắt đầu thực thi | **Trung bình:** kế hoạch sai sẽ cho kết quả sai dù mọi bước thực thi đúng |
| Khi output bất ngờ | Kết quả tool có cờ lỗi, rỗng, hoặc giá trị ngoài giới hạn mong đợi | **Biến thiên:** bắt các kiểu lỗi mà logic retry một mình không giải quyết được |

## 6. Điều phối tool: over-tooling và under-tooling

- Hành vi định tuyến của agent chịu hai yếu tố: cách mô tả tool và **số lượng tool đã đăng ký**.
- Quá nhiều tool với mô tả chồng lấn thì định tuyến thất thường. Quá ít tool thì agent hoặc ảo tưởng ra một lối đi, hoặc trả kết quả dở dang.
- **Over-tooling phổ biến hơn trong production:** nhóm đăng ký mọi tool "phòng khi cần" rồi thấy chất lượng chọn tool tụt khi bề mặt tool lớn dần. Cách làm: bắt đầu từ **bộ tối thiểu** cần cho nhiệm vụ, chỉ thêm tool khi xác nhận có lỗ hổng năng lực cụ thể.

## 7. Ràng buộc dữ liệu quy định quyết định route giao hàng và credentials trước khi viết wiring

- Nếu dữ liệu phải được xử lý theo ràng buộc cụ thể (đặc quyền luật sư - thân chủ, HIPAA, GDPR, FedRAMP, chính sách lưu trú dữ liệu nội bộ), ràng buộc đó quyết định **endpoint nào code gọi, mang credentials nào, và log đi về đâu**, trước mọi quyết định về prompt, tool hay bộ nhớ.
- Developer thường không chọn bề mặt, nhưng viết code nhắm vào endpoint cụ thể, gắn credentials, cấu hình region và phát log. Phải nêu rõ ràng buộc chi phối ngay từ đầu, vì cấu hình client sai tốn kém hơn nhiều để sửa sau khi agent đã nối xong.

Bảng năm ràng buộc (diễn giải, dựa trên những gì tôi đọc):

| Ràng buộc | Thường loại trừ gì trong code | Thường vượt qua review thế nào |
|---|---|---|
| Đặc quyền luật sư - thân chủ | Gọi từ bề mặt consumer mà công ty không audit được đầu-cuối; code gửi tài liệu đặc quyền tới endpoint chưa được duyệt | Gọi API/SDK trực tiếp từ ứng dụng của công ty, xác thực qua SSO, đi qua LLM gateway được duyệt có log đầy đủ request/response. Lưu ý: nội dung Compliance của Anthropic không mặc định được Anthropic thu trên lưu lượng API trực tiếp, nên tổ chức phải tự cài logging hội thoại ở lớp ứng dụng và đưa tới điểm lưu log được duyệt; chốt thiết kế log với account team |
| HIPAA (PHI) | Code gửi PHI tới endpoint hoặc route chưa nằm trong BAA, kể cả đường log/lưu giữ của chính code chưa nằm trong phạm vi BAA | API/SDK trực tiếp trên cấu hình có BAA (thỏa thuận với Anthropic), hoặc route qua AWS Bedrock/GCP Vertex trên tài khoản đủ điều kiện HIPAA sẵn có của đối tác. BAA không phủ Console, Workbench, tính năng beta hay gói consumer; không phải tính năng API nào cũng được bao phủ, kiểm tra danh sách hiện hành trong Implementation Guide |
| GDPR / lưu trú dữ liệu | Route mà region thực thi model không ghim được trong code, hoặc request có thể được phục vụ từ region ngoài ranh giới được duyệt; dùng endpoint global mà không chỉ định region là mẫu hỏng phổ biến | Route qua Bedrock/Vertex với region ghim trong cấu hình client. API Anthropic trực tiếp hiện **không** cung cấp lưu trú dữ liệu EU, nên đối tác có yêu cầu EU nên đi qua Bedrock hoặc Vertex |
| FedRAMP / chính phủ | Mọi đường code gọi endpoint không thuộc môi trường cloud được cấp phép ở mức tác động yêu cầu, kể cả đường dev/test chạm endpoint thương mại trong khi production chạm endpoint được cấp phép, vì credentials và mẫu code rò rỉ giữa hai bên | Ba route được cấp phép tại thời điểm bài: Claude for Government (C4G, FedRAMP High qua Palantir FedStart/PFCS-SS), Claude qua Amazon Bedrock GovCloud (FedRAMP High và DoD IL4/5), Claude qua Vertex AI Assured Workloads (FedRAMP). Claude Enterprise trên AWS Marketplace **không** được FedRAMP cấp phép. Kiểm tra trạng thái tại trust.anthropic.com |
| Chính sách lưu trú dữ liệu nội bộ | Mọi client SDK cấu hình trỏ vào cloud vendor ngoài danh sách được duyệt của đối tác, bất kể năng lực kỹ thuật có đáp ứng được hay không; ràng buộc mua sắm loại đường code trước khi sở thích kỹ thuật vào cuộc | Route giao hàng trên cloud vendor đã được duyệt: tức SDK client và cấu hình endpoint mà CIO đã thông qua. Build theo đó thay vì đổi giữa dự án chỉ vì route khác trông dễ hơn |

Hai ghi chú cuối bài: bảng này chỉ gồm các ràng buộc **quyết định chọn endpoint và cấu hình credentials**. **SOC 2 không thuộc phạm vi ở đây** vì nó chi phối cách hệ thống được xây và vận hành, không phải endpoint nào được gọi, và được học ở Module 4. Hộp "forward pointer" nói Module 4 đi sâu về IAM và quyền riêng tư theo thiết kế, phòng thủ prompt injection, guardrail lúc chạy và hardening agent; phần này chỉ có vai trò **nêu ràng buộc đúng lúc nó loại bỏ lựa chọn**: khi chọn endpoint, cấu hình SDK client và credentials mang theo vào production.

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt**

- Trước tiên hỏi: có cần agent không? Liệt kê được các bước thì dùng workflow.
- Vòng lặp giống nhau ở cả ba đường (raw API, Agent SDK, Managed Agents); khác ở lượng hạ tầng bạn tự giữ.
- Managed Agents hiện chưa đủ điều kiện ZDR/HIPAA, nên với PHI hoặc ZDR thì loại ngay.
- Bốn bước nối vòng lặp: đăng ký tool, system prompt có phạm vi, xử lý tool-use loop, điều kiện thoát. Cộng thêm điểm HITL cho hành động không hoàn tác.
- Ít tool nhưng đúng tốt hơn nhiều tool chồng lấn. Ràng buộc dữ liệu quy định quyết định endpoint và credentials trước khi viết wiring.

**ELI5**

Có ba cách đi từ nhà đến sân bay: tự lái xe (raw API: bạn lo mọi thứ), đi xe công nghệ có bạn ngồi cạnh dặn đường (Agent SDK: khung lái có sẵn, bạn vẫn xử lý việc thực tế), hoặc thuê xe đưa đón trọn gói (Managed Agents: tiện nhưng chạy theo luật của hãng). Nếu hành lý là thứ không được phép qua hãng đó (PHI/ZDR), thì dù tiện đến mấy cũng không chọn được.

**Ví dụ khi implement**

*Snippet 1: `settingSources` nên đặt tường minh (Agent SDK)*

```python
options = ClaudeAgentOptions(setting_sources=["user", "project", "local"])  # giống Claude Code CLI
# hoặc [] để chạy cô lập hoàn toàn
```

Dùng để: không phụ thuộc mặc định khi nạp CLAUDE.md và skill. Tên tham số theo từng SDK có thể khác, đối chiếu tài liệu Agent SDK.

*Snippet 2: xử lý mọi tool_use của cùng một lượt rồi trả đủ tool_result*

```python
results = [
    {"type": "tool_result", "tool_use_id": b.id, "content": run_tool(b.name, b.input)}
    for b in resp.content if b.type == "tool_use"
]
messages.append({"role": "user", "content": results})
```

Dùng để: không bỏ sót tool_use nào trong lượt (mục 3 của checklist nối vòng lặp).

*Snippet 3: điều kiện thoát không dựa vào việc Claude tự nguyện dừng*

```python
for _ in range(MAX_ITERATIONS):
    resp = client.messages.create(...)
    if resp.stop_reason != "tool_use":
        break
```

Dùng để: luôn có giới hạn số vòng và tiêu chí dừng rõ ràng.
