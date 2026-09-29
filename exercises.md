# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết phần phản ánh trực tiếp dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Hồng  Mã học viên: 2A202602800

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên Railway, nếu tôi quên tạo `AGENT_API_KEY`, `Settings` sẽ ném
`ValidationError` ngay lúc process khởi động. Deployment vì thế bị phát hiện là
chưa đủ cấu hình trước khi nhận traffic. Nếu để mặc định `"changeme"`, service
vẫn có thể báo healthy và vô tình chấp nhận một khóa mà mọi người đều biết; tôi
chỉ phát hiện lỗi sau khi API đã được mở công khai. Fail fast làm lỗi cấu hình
thành lỗi triển khai rõ ràng, thay vì thành sự cố bảo mật âm thầm.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng JSON tôi quan sát được khi gọi logic ghi log là:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:26:56.581856+00:00", "user_id": "cp5-observation", "tokens_in": 17, "tokens_out": 23, "cost_usd": 0.0004}
```

Thứ nhất, tôi có thể lọc và đếm tất cả event `ask_completed` theo `user_id`,
thời gian hoặc level để tìm request lỗi và lập cảnh báo. Thứ hai, tôi có thể
cộng `tokens_in`, `tokens_out` và `cost_usd` để theo dõi mức sử dụng và chi phí
theo user hoặc theo khoảng thời gian. Một câu `print("đã trả lời xong")` không có
schema, timestamp hay các trường định lượng nên hệ thống log khó truy vấn và
không thể tổng hợp đáng tin cậy như vậy.

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
| 1 stage (bản đầu, build thử) | 63,861,880 bytes ≈ 60.9 MB |
| Multi-stage | 63,861,503 bytes ≈ 60.9 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Tôi đo bằng `docker image inspect`, không làm tròn theo dòng hiển thị của
`docker images`. Hai image trong bài này gần như bằng nhau, chỉ chênh 377
bytes, vì runtime đều dùng `python:3.11-slim` và các dependency hiện tại có
thể cài trực tiếp mà không kéo theo một bộ compiler lớn. Vì vậy không nên bịa
ra một mức giảm hàng trăm MB cho kết quả này.

Tuy nhiên multi-stage vẫn có ý nghĩa: dependency được cài ở stage `builder`
rồi chỉ thư mục `/install` được chép sang runtime. Nếu stage build cần compiler,
header hoặc công cụ biên dịch native, các thứ đó sẽ không đi vào image runtime;
image cuối chỉ giữ Python, dependency runtime và source cần chạy. Chênh lệch
thực tế phụ thuộc dependency, còn lợi ích chính là tách rõ build artifact khỏi
runtime artifact.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại, khi chỉ sửa một ký tự trong `app/main.py`, base image,
`WORKDIR`, `COPY requirements.txt` và `RUN pip install` ở stage `builder` vẫn
được lấy từ cache vì requirements không đổi. Stage runtime cũng dùng lại base
image và phần dependency đã copy từ builder. Layer `COPY app ./app` phải chạy
lại; do Docker cache có tính dây chuyền, các layer đứng sau nó như `COPY utils`,
tạo user và metadata cuối cũng được dựng lại, dù việc đó thường rất nhanh.

Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source đều làm layer
copy thay đổi. Khi đó Docker phải chạy lại `pip install` và toàn bộ layer sau
nó. Cách đặt requirements trước source giúp sửa code mà không phải cài lại
dependency, nên build nhanh và ổn định hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi rủi ro là: một lỗ hổng cho phép kẻ tấn công thực thi Python hoặc lệnh
shell trong process của agent; nếu process đó chạy bằng root thì kẻ tấn công có
quyền cao trong container, có thể đọc secret, sửa file và truy cập các dịch vụ
nội bộ. Nếu Docker daemon, mount hoặc kernel có cấu hình yếu, quyền root trong
container còn làm tăng khả năng leo thang hoặc thoát container để tác động lên
host.

`USER appuser` trong Dockerfile cắt chuỗi này ngay sau bước khai thác: process
ứng dụng chỉ có quyền của user không đặc quyền, không thể tùy ý sửa hệ thống
hay thực hiện các thao tác chỉ dành cho root. Đây không phải là thay thế cho
việc vá lỗ hổng và giới hạn network, nhưng nó giảm đáng kể hậu quả khi lỗi
ứng dụng bị khai thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Hạn mức là 10 request/phút. Với bộ đếm reset theo phút đồng hồ, người dùng có
thể gửi 10 request ngay trước mốc chuyển phút, ví dụ ở giây 59, rồi gửi thêm 10
request ngay sau mốc ở giây 00. Như vậy có tối đa **20 request trong khoảng 2
giây**, dù mỗi bucket riêng lẻ chỉ có 10. Sliding window nhìn vào đúng 60 giây
gần nhất nên tại thời điểm sau mốc chuyển phút, 10 request cũ vẫn còn trong cửa
sổ và request mới sẽ bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tần suất hoặc số lượng request trong một cửa sổ thời gian;
nó bảo vệ CPU, Redis và độ ổn định của API. Cost guard giới hạn số tiền đã tiêu
trong tháng, cộng cả chi phí ước tính của request mới; nó bảo vệ ngân sách.

Ví dụ rate limit cho qua nhưng cost guard chặn: user mới chỉ gửi 8 request,
chưa vượt 10 request/phút, nhưng mỗi request tạo prompt rất lớn nên tổng chi
phí đã gần hết ngân sách tháng. Request tiếp theo phải trả `402` trước khi gọi
LLM. Chiều ngược lại, user còn rất nhiều ngân sách nhưng gửi 11 request rẻ
trong một phút; rate limiter chặn request thứ 11 bằng `429`, dù cost guard vẫn
còn cho phép về mặt tiền bạc. Hai cơ chế vì thế bảo vệ hai tài nguyên khác
nhau và cần chạy trước bước gọi LLM.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp hai probe và probe đó kiểm tra Redis, thứ tự sẽ là:

1. Redis mất kết nối; probe của cả ba container bắt đầu timeout hoặc nhận lỗi.
2. Cả ba container cùng trả trạng thái không khỏe, dù process Python vẫn đang
   sống và `/health` nhẹ lẽ ra vẫn có thể trả lời.
3. Load balancer/orchestrator loại cả ba instance khỏi traffic hoặc khởi động
   lại chúng. Trong khoảng này cụm không còn instance phục vụ, và việc restart
   đồng loạt có thể tạo thêm lỗi/cascade.
4. Sau 30 giây Redis hoạt động lại, các container mới phải khởi động và kết nối
   lại; traffic chỉ trở lại sau khi probe thành công.

Thiết kế hiện tại tách hai việc: `/health` chỉ kiểm tra process nên không bị
restart oan khi Redis chập chờn, còn `/ready` kiểm tra `store.ping()` và trả
`503` để load balancer tạm ngừng gửi request. Khi Redis phục hồi, `/ready`
trả `200` và instance có thể nhận traffic lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis, các instance đều đọc cùng một danh sách nên lịch sử không phụ thuộc
request rơi vào container nào. Trong code hiện tại, `history_length` được lấy
trước khi ghi lượt hỏi và lượt trả lời: request đầu thường thấy `0`, request
tiếp theo thấy `2`, rồi `4`, và tiếp tục tăng cho đến giới hạn 20 message.

Nếu dùng một `dict` Python riêng trong mỗi process, mỗi container có một bản
nhớ khác nhau. Khi load balancer chuyển request sang instance mới, response có
thể quay lại `history_length=0` hoặc một giá trị ngắn hơn; nếu quay lại đúng
instance cũ thì nó mới thấy lịch sử của instance đó. Sau khi container restart,
toàn bộ dict cũng mất. Vì vậy kết quả sẽ không đều và hội thoại bị mất, trong
khi Redis cung cấp state dùng chung cho cả ba instance.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi thực tế tôi gặp là Redis add-on ban đầu trên Railway ở trạng thái hỏng,
nên service agent không có một Redis usable để kết nối. Tôi phát hiện bằng cách
kiểm tra trạng thái/deployment của hai service và gọi endpoint readiness: khác
với `/health` chỉ kiểm tra process, `/ready` phụ thuộc Redis nên không đạt khi
Redis không ping được.

Tôi đã xóa add-on bị hỏng, tạo service `redis-runtime` chạy image
`redis:7-alpine` trên private network, rồi cập nhật `REDIS_URL` của agent tới
hostname nội bộ của service mới. Sau khi redeploy, kiểm tra public cho kết quả
`/health` là `200`, `/ready` là `200` với `redis: true`, và `/ask` không có key
trả `401` còn request có key hợp lệ trả `200`. Cách sửa này cũng giữ secret ở
Railway Variables, không chép giá trị secret vào repository.
