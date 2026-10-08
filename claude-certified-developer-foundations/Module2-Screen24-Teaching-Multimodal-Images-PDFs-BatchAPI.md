---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Multimodal and batch ingestion
screen: 24 of 29
screen_type: TEACHING (khoảng 13 phút)
topic: Multimodal and batch ingestion
screen_title: Images, PDFs, and high-volume processing
related_screens: Screen 13-14 (context budget), Screen 19 (Files API/skills ở mức khái niệm)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần đầu bám đủ nội dung màn gốc, phần "Tóm tắt, ELI5 và ví dụ" ở cuối là phần Claude thêm
---

# [Module 2 · Screen 24 · TEACHING] Ảnh, PDF và xử lý khối lượng lớn

## Ý chính của màn

Các màn trước nói về việc Claude nhớ gì giữa các lượt. Màn này chuyển sang câu hỏi **bạn gửi gì vào**: mỗi ảnh và mỗi PDF đều **tiêu tốn ngân sách context trước khi Claude đọc chữ nào trong prompt**, nên nó thay đổi cách bạn cấu trúc request và lượng bạn nhét được vào một request. Nửa sau của màn nói về đầu kia của cùng vấn đề: khi có hàng nghìn input, gửi từng request một rồi chờ là không hợp lý, và **Batch API** là cách xử lý khối lượng đó mà không chặn ứng dụng.

## Chi phí token của ảnh: tính trước khi cam kết

- Ảnh không miễn phí về context. Claude nhìn ảnh theo **patch**: mỗi khối **28×28 pixel** là một visual token, nên ảnh tốn khoảng **[rộng / 28] × [cao / 28]** token.
- Ví dụ: ảnh 1.000 × 1.000 px là [1000/28] × [1000/28] ≈ 36 × 36 patch, khoảng **1.296 visual token**. Với tốc độ đó, mười ảnh chụp màn hình độ phân giải cao tốn context ngang một system prompt chi tiết.
- Mỗi model có **độ phân giải gốc tối đa** (giới hạn cạnh dài và giới hạn visual token), và giới hạn này **khác nhau theo tier**: model mới nhất nhận ảnh lớn hơn đáng kể so với tier chuẩn. Ảnh vượt giới hạn sẽ bị **thu nhỏ trước khi xử lý**, nên công thức chạy trên kích thước sau khi thu nhỏ.
- Hãy đối chiếu giới hạn từng tier với trang Vision của tài liệu (mục Resolution and token cost) **lúc build**, vì giới hạn đã đổi giữa các thế hệ model và sẽ còn đổi.
- Việc tính này quan trọng ở **thời điểm thiết kế**: đo token của một ảnh production điển hình so với giới hạn context của model **trước khi viết code ingest**. Một pipeline vượt ngân sách thường chỉ cần một bước resize mười phút để sửa; phát hiện sau khi triển khai thì tốn lâu hơn nhiều.

## Các cách gửi ảnh: khi nào dùng cách nào

Trang có ba tab, mỗi tab nêu cách hoạt động, chi phí phụ và lúc nên dùng.

| Cách | Hoạt động | Chi phí phụ | Khi nào dùng |
|---|---|---|---|
| **Inline base64** | Mã hóa byte ảnh thành chuỗi base64 và đưa thẳng vào message block | Toàn bộ payload đã mã hóa đi theo **mỗi request**, làm request to ra và tăng latency với ảnh lớn | Ảnh dùng một lần, nơi thêm bước upload chỉ làm phức tạp mà không có lợi. Cùng một ảnh gửi lặp lại sẽ nhân chi phí lên, nên nếu có khả năng dùng lại thì chọn cách khác |
| **URL reference** | Truyền một URL công khai vào source block, Claude tải ảnh lúc nhận request | Không có payload đi kèm, nhưng bạn phụ thuộc vào việc URL phải **ổn định, công khai và truy cập được đúng lúc Claude tải** | Ảnh đã nằm sẵn ở một URL công khai ổn định do bạn kiểm soát. Bỏ qua nếu ảnh nằm sau xác thực, URL ký có hạn ngắn, hoặc bạn không đảm bảo truy cập được khi request chạy |
| **Files API** | Upload file một lần qua một API call riêng, nhận về `file_id`, rồi tham chiếu ID đó ở mọi message sau | Upload là chi phí một lần; mỗi request sau chỉ mang ID thay vì byte nên overhead payload gần bằng 0. Hiện ở **beta** và **không có trên Bedrock hay Vertex AI**, cần kiểm tra nền tảng triển khai của bạn | Cùng một ảnh hoặc PDF xuất hiện ở nhiều request, hoặc asset đủ lớn để gửi lại sẽ chiếm phần lớn kích thước request. Cũng là lựa chọn gọn nhất khi muốn tách quản lý asset khỏi các call inference, và đúng cho ảnh xuất hiện qua nhiều lượt hội thoại, vì `file_id` không làm history nặng thêm |

## Gửi PDF: block `document`

- Với PDF, loại block là **`document`** thay vì `image`. Cấu trúc `source` theo cùng mẫu với ảnh: có thể là base64, URL hoặc `file_id` của Files API.
- Block `document` **không có trường `name` bắt buộc**. Nó nhận trường `title` (tên đọc được của tài liệu) và trường `context` (metadata bổ sung), nhưng không trường nào bắt buộc để gửi PDF.
- Mọi cơ chế còn lại, kể cả cân nhắc về chi phí token và việc tái sử dụng qua Files API, áp dụng giống như với ảnh.

Hình dạng block mà trang đưa ra (viết lại gọn):

```json
{
  "type": "document",
  "source": {
    "type": "base64",
    "media_type": "application/pdf",
    "data": "<base64-encoded-pdf-bytes>"
  },
  "title": "contract_review.pdf"
}
```

## Áp dụng kỹ thuật prompting cho đầu vào đa phương thức

- Các kỹ thuật prompting ở phần đầu khóa **áp dụng nguyên cho phân tích ảnh và PDF**. Một prompt trần kiểu "describe this image" cho kết quả nông, giống như prompt văn bản trần, vì Claude không có cấu trúc đích để hướng tới.
- Khác biệt là ảnh mang **tính mơ hồ mà văn bản không có**: vật thể chồng lên nhau, độ sâu và quan hệ không gian, bị che khuất một phần. Prompt cho phân tích ảnh nên nêu rõ Claude xử lý từng loại mơ hồ ra sao.
- Ví dụ của trang: "Nếu các vật thể chồng lên nhau, hãy mô tả từng vật riêng và ghi chú sự chồng lấn" là một ràng buộc cụ thể mà prompt chỉ có văn bản sẽ không bao giờ cần.

## Message Batches API: xử lý bất đồng bộ khối lượng lớn

- Khi cần chạy cùng một mẫu prompt trên hàng trăm hoặc hàng nghìn input, API đồng bộ là mô hình sai: mỗi call chặn cho tới khi xong, nên ở quy mô lớn ứng dụng hoặc đốt thread, hoặc mở hàng nghìn kết nối đồng thời va vào rate limit.
- Message Batches API nhận tới **100.000 request hoặc 256 MB** (cái nào chạm trước) trong một batch call. Bạn gửi batch, nhận `batch_id`, **poll** tới khi xong, rồi tải kết quả. **Giá theo token của batch thấp hơn** API đồng bộ.
- Đánh đổi là **latency**: batch không có thời gian xác định, có thể tới **24 giờ**, thường nhanh hơn nhiều. Mẫu này hợp với pipeline offline, chạy đánh giá và job xử lý dữ liệu, **không hợp với tương tác thời gian thực của người dùng**.

| Tình huống | API đúng | Lý do |
|---|---|---|
| Người dùng upload ảnh và chờ phân loại ngay | Đồng bộ | Cần phản hồi realtime; latency của batch không chấp nhận được cho tương tác |
| Pipeline ban đêm phân loại 5.000 bản ghi khách hàng | Message Batches | Latency không phải ràng buộc; cả giảm chi phí và xử lý bất đồng bộ đều có giá trị |
| Chạy đánh giá prompt mới trên 2.000 ví dụ | Message Batches | Tác vụ offline, không có yêu cầu realtime |
| Chatbot tạo câu trả lời cho tin nhắn của người dùng | Đồng bộ | Người dùng đang chờ; batch gây trễ không chấp nhận được |

## Khi nào multimodal và batch ăn khớp, khi nào không

- **Ăn khớp** với workload offline tái sử dụng cùng asset và cần output có cấu trúc trên hàng nghìn input. Ví dụ điển hình: pipeline ban đêm phân loại ảnh theo một taxonomy cố định, trong đó Files API bỏ việc upload lặp, Batches API hấp thụ latency, và các kỹ thuật structured output giữ kết quả đọc được bằng máy.
- **Hai kiểu hỏng** phá sự ăn khớp này:
  1. **Hiểu sai latency:** dùng batch trong luồng hướng người dùng có ảnh sẽ tạo ra hệ thống qua được test nhưng hỏng ở production, vì người dùng đang chờ còn batch thì không.
  2. **Đánh giá thấp chi phí context:** ảnh và PDF ngốn ngân sách trước khi Claude xử lý chữ nào, nên pipeline nạp nhiều ảnh lớn mỗi request sẽ vượt giới hạn token khi lên quy mô. Hãy đo chi phí token trên đầu vào quy mô production trước khi build.

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt:** ảnh và PDF tốn context theo kích thước (ảnh: số patch 28×28), nên đo trước. Chọn cách gửi theo mức tái sử dụng: base64 cho một lần, URL cho ảnh công khai ổn định, Files API khi dùng lại nhiều lần. Dùng Batch API cho việc offline số lượng lớn, đồng bộ cho việc người dùng đang chờ.

**ELI5:** gửi ảnh giống gửi bưu kiện. Ảnh to thì tốn nhiều chỗ trên xe (context), nên cân trước khi gửi. Gửi base64 là mang cả bưu kiện theo mỗi chuyến; URL là chỉ đường cho shipper tự đến lấy (đường phải còn mở); Files API là gửi bưu kiện vào kho một lần rồi chỉ ghi mã kho. Batch giống gom cả xe hàng gửi đêm, rẻ hơn nhưng không giao ngay; khách đang đứng chờ ở quầy thì phải giao tay.

**Snippet 1: tính nhanh số token ảnh**

```python
def image_tokens(w, h, patch=28):
    return (w // patch) * (h // patch)

image_tokens(1000, 1000)  # 36 * 36 = 1296
```

Dùng để: ước lượng ngân sách context của pipeline ảnh trước khi viết code ingest (chỉ là ước lượng theo công thức của trang, chưa tính việc ảnh bị thu nhỏ khi vượt giới hạn tier).

**Snippet 2: tái sử dụng PDF qua Files API**

```python
doc_block = {
    "type": "document",
    "source": {"type": "file", "file_id": file_id},
    "title": "contract_review.pdf",
}
```

Dùng để: upload một lần, các request sau chỉ mang `file_id`. Dạng `source` cho file_id là phần Claude viết theo mẫu "cùng cấu trúc với ảnh" của trang; trang không in snippet này, và Files API đang beta nên cần kiểm tra tài liệu hiện hành trước khi dùng.

**Snippet 3: gửi batch và poll**

```python
batch = client.messages.batches.create(requests=requests)  # tối đa 100k request / 256 MB
batch_id = batch.id
# ... poll cho tới khi batch.processing_status == "ended", rồi tải kết quả
```

Dùng để: chạy việc offline số lượng lớn rẻ hơn, chấp nhận chờ. Tên method và trường là phần Claude viết theo mô tả "gửi, nhận batch_id, poll, tải kết quả" của trang; trang không in code này.
