# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trả lời mẫu dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hoang Phong  Mã học viên: 2A202602943

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là tôi deploy image lên cloud nhưng quên tạo biến
> `AGENT_API_KEY` trên dashboard. Vì trường này bắt buộc, Pydantic báo lỗi
> validation ngay khi ứng dụng đọc Settings và deployment không chuyển sang
> trạng thái healthy. Tôi có thể nhìn build/runtime log và sửa cấu hình trước
> khi public service. Nếu code dùng mặc định `"changeme"`, app vẫn khởi động
> và health check vẫn xanh; bot hoặc người biết khóa mặc định có thể gọi
> `/ask`, tiêu quota và ngân sách trước khi tôi phát hiện cấu hình sai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được khi gọi `/ask` là:
>
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T11:06:35.606277+00:00", "user_id": "cp4-validation-20260929", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`
>
> Từ log JSON này, hệ thống logging có thể lọc hoặc đếm sự kiện
> `ask_completed` theo `user_id` và khoảng thời gian để điều tra hành vi của
> một user. Nó cũng có thể cộng `tokens_in`, `tokens_out`, `cost_usd` để dựng
> dashboard chi phí hoặc cảnh báo khi chi phí tăng bất thường. Dòng
> `print("đã trả lời xong")` không có timestamp hay field có cấu trúc nên
> không thực hiện đáng tin cậy được hai việc đó.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại cả hai image và đọc kích thước bằng `docker images`. Bản đầu
> dùng image `python:3.11` đầy đủ nên chứa nhiều thư viện hệ thống và công cụ
> không cần cho lúc chạy. Bản mới dùng `python:3.11-slim`; dependency được cài
> ở stage `builder`, sau đó chỉ thư mục kết quả `/install` được chép sang stage
> runtime. Vì stage builder không đi vào image cuối, bản runtime không mang
> theo các thành phần chỉ phục vụ build. Kết quả giảm từ 1.73 GB xuống 271 MB,
> tức giảm khoảng 1.46 GB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sau khi thay đổi một file trong `app/` và build lại, log BuildKit cho thấy
> các layer base image, `WORKDIR`, `COPY requirements.txt` và `RUN pip install`
> đều dùng lại cache. Layer `COPY app ./app` phải chạy lại vì source đã đổi;
> các layer đứng sau nó cũng phải được tạo lại. Dockerfile hiện tại tách
> `requirements.txt` khỏi source nên sửa code không làm cài lại thư viện. Nếu
> đặt `COPY . .` trước `RUN pip install`, bất kỳ thay đổi nào trong source cũng
> làm layer `COPY` đổi, kéo theo cache của `pip install` bị vô hiệu và toàn bộ
> dependency phải được cài lại.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng như command injection hoặc unsafe deserialization có thể cho kẻ
> tấn công thực thi lệnh với đúng quyền của process Python. Nếu process chạy
> bằng root trong container, họ có thể sửa mọi file trong container, cài thêm
> công cụ và khai thác các cấu hình nguy hiểm như volume nhạy cảm, Docker
> socket, Linux capability dư thừa hoặc một lỗ hổng container escape để tác
> động đến host. `USER appuser` chuyển process sang UID 10001 trước khi chạy
> Uvicorn, nên quyền root bị cắt ngay tại bước thực thi lệnh: kẻ tấn công chỉ
> nhận quyền của user hạn chế. Biện pháp này không thay thế việc vá lỗ hổng hay
> cấu hình container an toàn, nhưng làm giảm đáng kể phạm vi thiệt hại nếu app
> bị chiếm quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Cách đếm theo phút đồng hồ có thể cho qua tối đa 20 request trong 2 giây:
> user gửi 10 request vào khoảng 10:00:59, bộ đếm reset ở 10:01:00, rồi gửi
> tiếp 10 request vào khoảng 10:01:01. Mỗi phút riêng lẻ vẫn chỉ ghi nhận 10
> request nhưng tải thực tế là một burst 20 request. Với sliding window, lần
> gọi ở 10:01:01 vẫn nhìn lại 60 giây gần nhất và thấy 10 request cũ nên bị
> chặn. Trong test của tôi, ba request đầu được phép khi limit là 3 và request
> thứ tư nhận HTTP 429; sau khi các timestamp cũ ra khỏi cửa sổ thì request lại
> được phép.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit kiểm soát tốc độ/số request trong một khoảng thời gian ngắn, còn
> cost guard kiểm soát tổng số tiền tích lũy của từng user trong cả tháng. Ví
> dụ rate limit có thể cho qua một user chỉ gửi 2 request/phút, nhưng cost guard
> phải chặn nếu các request trước đã làm họ dùng hết ngân sách tháng hoặc mỗi
> request dùng rất nhiều token. Ngược lại, một user mới gần như chưa tốn ngân
> sách có thể gửi một burst vượt 10 request trong 60 giây: cost guard vẫn còn
> tiền nên cho qua, nhưng rate limiter phải trả 429. Test cũng xác nhận hai user
> có quota và ngân sách riêng, và trường hợp vượt ngân sách trả HTTP 402.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp hai endpoint và bắt health check phụ thuộc Redis, trình tự sẽ là:
> Redis mất kết nối, cả ba container cùng kiểm tra thất bại và trả 503; load
> balancer ngừng gửi traffic, đồng thời orchestrator coi cả ba process là hỏng
> và restart chúng. Các container mới vẫn chưa kết nối được Redis nên tiếp tục
> fail health check và rơi vào vòng lặp restart. Khi Redis quay lại sau 30 giây,
> cả cụm vẫn cần khởi động lại và warm up nên thời gian gián đoạn bị kéo dài.
> Nếu tách riêng, `/health` vẫn trả 200 vì process còn sống, còn `/ready` trả
> 503 để tạm rút instance khỏi traffic mà không restart hàng loạt. Test của tôi
> xác nhận Redis lỗi làm `/ready` trả 503 nhưng không làm `/health` phụ thuộc
> Redis.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi chạy kiểm tra với Redis thật, hai request liên tiếp của cùng user trả
> `history_length` lần lượt là 0 và 2. Test mô phỏng hai instance
> `ConversationStore` riêng dùng chung Redis cũng cho thấy instance B đọc được
> message mà instance A vừa ghi. Vì mỗi lượt thêm một message `user` và một
> message `assistant`, nếu gọi năm lần qua các replica thì history dùng chung sẽ
> tăng theo 0, 2, 4, 6, 8 bất kể request vào container nào. Nếu dùng một dict
> Python riêng cho từng container và request được chia vòng qua ba container,
> tôi sẽ thấy dạng 0, 0, 0, 2, 2 thay vì tăng liên tục; với cân bằng tải ngẫu
> nhiên, con số còn có thể lặp hoặc nhảy thất thường. Restart một container
> cũng làm riêng lịch sử trong dict của container đó biến mất.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi Railway báo deploy thành công, lần kiểm tra đầu tiên của tôi thất bại
> với lỗi `Invoke-RestMethod: Invalid URI: The hostname could not be parsed`.
> Tôi kiểm tra lại biến `$url` và nhận ra mình đã dán lệnh khi PowerShell còn ở
> dấu nhắc tiếp dòng `>>`, nên giá trị URL bị ghép sai với phần văn bản khác.
> Tôi nhấn `Ctrl+C`, lấy lại domain thật bằng `railway domain`, gán chính xác
> `https://agent-production-fcad.up.railway.app` và gọi endpoint bằng cú pháp
> `${url}/health`. Sau khi sửa, `/health` trả 200 với `status=ok`, `/ready` trả
> 200 với `redis=true`, request `/ask` thiếu key trả 401 và request có key hợp
> lệ trả 200.
