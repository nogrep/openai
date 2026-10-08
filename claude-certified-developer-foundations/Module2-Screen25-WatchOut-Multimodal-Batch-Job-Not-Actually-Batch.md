---
course: Production-Grade Prompting, Agents & Tool Use
series: Claude Certified Developer - Foundations (Prep Course)
module: 2 - Production-Grade Prompting, Agents & Tool-use
section: Multimodal and batch ingestion
screen: 25 of 29
screen_type: WATCH OUT (lỗi thường gặp, khoảng 4 phút)
topic: Multimodal and batch ingestion
screen_title: The batch job that was not actually a batch
related_screens: Screen 24 (Teaching - ảnh, PDF và Message Batches API)
note: Ghi chú diễn giải bằng lời của Claude, không phải văn bản gốc của khóa học. Phần đầu bám đủ nội dung màn gốc, phần "Tóm tắt, ELI5 và ví dụ" ở cuối là phần Claude thêm
---

# [Module 2 · Screen 25 · WATCH OUT] Batch job mà thật ra không phải batch

## Bối cảnh (phần "Setup")

Chia một job thành từng chunk rồi xử lý lần lượt **không phải là batching**, mà là **tuần tự hóa thêm vài bước**. Message Batches API tồn tại cho workload khối lượng lớn chính vì việc lặp qua từng input trên API đồng bộ sẽ **đụng rate limit ngay khi khối lượng thành thật**, dù bạn cắt danh sách input thế nào.

## Case study: cuộc trò chuyện nội bộ về job ban đêm liên tục dính rate limit

Một developer chạy lại cùng một job phân loại ban đêm suốt **ba đêm** và lần nào cũng dính lỗi rate limit ở khoảng cùng một chỗ. Developer cấp cao hỏi một câu để lộ ra vấn đề thật.

- **Developer:** job ban đêm cứ dính rate limit; mình đã chia nhỏ thành các chunk rồi, còn làm gì được nữa?
- **Senior:** em gửi chúng bằng cách nào?
- **Developer:** em lặp qua danh sách và gọi API cho từng item.
- **Senior:** đó không phải batching, đó là các call nối đuôi nhau lên endpoint đồng bộ. Chia danh sách thành chunk không đổi điều API nhìn thấy: nó vẫn thấy **một request cho mỗi item, nối tiếp nhau**.
- **Developer:** vậy rate limit nổ vì em đang thực hiện hàng nghìn call đồng bộ?
- **Senior:** đúng. Message Batches API nhận tới **100.000 request hoặc 256 MB** mỗi batch trong một call, trả về `batch_id` và xử lý **bất đồng bộ**. Em **poll** để biết khi nào xong, nghĩa là code kiểm tra trạng thái batch theo lịch cho tới khi API báo xong. Giá theo token thấp hơn đồng bộ, và **rate limit không nổ** vì em không tạo ra hàng nghìn request riêng lẻ.
- **Developer:** còn đánh đổi?
- **Senior:** latency không xác định. Batch có thể mất hàng giờ. Nếu đây là tương tác thời gian thực với người dùng thì là sai công cụ, nhưng với job phân loại ban đêm thì hoàn hảo.

## Điều cần nhớ (phần "What to watch out for")

1. **Chia chunk rồi lặp qua API đồng bộ không phải batching**, dù cảm giác như nó phải là. Nó tạo ra **cùng số API call** như bản không chia chunk và dính **cùng rate limit**.
2. **Message Batches API là một mô hình gửi khác, không phải "batch size nhỏ hơn".** Dùng nó bất cứ khi nào workload **khối lượng lớn và offline**; chỉ dùng API đồng bộ khi **có người dùng đang chờ ở đầu kia**.
3. **Kết quả trả về theo thứ tự tùy ý**, không theo thứ tự đã gửi. Dùng trường **`custom_id`** trên mỗi request để ghép kết quả về đúng input.

## Tóm tắt, ELI5 và ví dụ (phần Claude thêm)

**Tóm tắt một câu:** muốn xử lý số lượng lớn offline thì nộp cả cục qua Batches API và ghép kết quả bằng `custom_id`; cắt nhỏ rồi gọi đồng bộ từng cái vẫn là cùng số call và vẫn dính rate limit.

**ELI5:** bạn cần gửi 5.000 lá thư. Cắt chồng thư thành 50 xấp rồi ra hòm thư bỏ từng lá một thì vẫn là 5.000 lần bỏ thư, và bác bưu tá vẫn phàn nàn như cũ. Batch API là đưa cả bao thư cho bưu cục, họ xử lý khi rảnh, rẻ hơn nhưng không báo trước giờ nào xong. Thư về không theo thứ tự gửi, nên mỗi lá phải dán số thứ tự riêng (`custom_id`).

**Snippet 1: kiểu "giả batch" (tránh)**

```python
for chunk in chunks(records, 100):
    for r in chunk:
        client.messages.create(...)  # vẫn là 1 request/item, vẫn dính rate limit
```

Dùng để: nhận ra mẫu sai. Chunk chỉ đổi cách nhóm trong code của bạn, không đổi điều API nhìn thấy.

**Snippet 2: nộp batch với `custom_id`**

```python
requests = [
    {"custom_id": f"rec-{r['id']}", "params": {...}}  # params = tham số messages.create
    for r in records
]
batch = client.messages.batches.create(requests=requests)
```

Dùng để: một lần nộp cho cả tập input, mỗi request mang `custom_id` để ghép lại sau. Hình dạng request là phần Claude viết theo mô tả của trang; trang không in code này.

**Snippet 3: ghép kết quả theo `custom_id`**

```python
by_id = {item.custom_id: item.result for item in results}
label = by_id[f"rec-{record_id}"]
```

Dùng để: kết quả về không theo thứ tự gửi, nên tra theo `custom_id` thay vì theo vị trí. Tên trường kết quả là phần Claude viết theo mô tả của trang.
