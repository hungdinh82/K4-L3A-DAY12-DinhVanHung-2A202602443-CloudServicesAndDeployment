# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: mỗi mục bên dưới đã được thay bằng câu trả lời tương ứng.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đinh Văn Hùng  Mã học viên: 2A202602443

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy, nếu quên khai báo `AGENT_API_KEY` mà app vẫn dùng `"changeme"`, bất kỳ ai đoán được giá trị này đều có thể gọi `/ask`. Với fail fast, process khởi động thất bại ngay và Render không đưa một service cấu hình sai ra nhận traffic; tôi phát hiện thiếu biến môi trường trước khi có request nào đi vào API.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log khi xử lý request có dạng: `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:23:28+00:00", "user_id": "local-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}`. Tôi có thể lọc tất cả request của một `user_id` để điều tra lỗi và cộng/truy vấn trường `cost_usd` để theo dõi chi phí. Chuỗi `print("đã trả lời xong")` không có trường dữ liệu có cấu trúc để máy lọc hay tổng hợp.

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
| Multi-stage | 297 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản một stage dùng base `python:3.11` đầy đủ, còn bản multi-stage dùng `python:3.11-slim` và chỉ mang các dependency đã cài sang runtime. Phần chênh lệch chủ yếu là công cụ/hệ thống của image đầy đủ và những thành phần build không cần để chạy API. Image đa stage cũng không copy toàn bộ build context vào runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, layer base image, `WORKDIR`, `COPY requirements.txt` và `RUN pip install` vẫn được cache. Chỉ layer copy source code và các layer sau nó chạy lại. Nếu `COPY . .` đứng trước `RUN pip install`, chỉ một ký tự đổi trong source cũng làm mất cache của layer copy, khiến `pip install` chạy lại dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Ví dụ một endpoint có lỗ hổng cho phép thực thi lệnh hệ điều hành. Nếu process trong container chạy root, mã lệnh đó có toàn quyền trong container: đọc/sửa file ứng dụng, cài công cụ và tìm cách khai thác mount, socket Docker hoặc cấu hình sai để leo thang sang host. `USER appuser` làm process Uvicorn và lệnh bị thực thi chỉ có quyền UID 10001, nên không thể sửa các vị trí chỉ root được phép hay trực tiếp có đặc quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi 20 request trong gần 2 giây: gửi 10 request ở giây 59 của một phút, sau đó gửi 10 request tiếp ở giây 00 của phút kế tiếp. Bộ đếm theo phút đồng hồ reset ở mốc phút mới nên cho qua cả hai nhóm. Sliding window 60 giây vẫn nhìn thấy 10 request đầu và chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit kiểm soát tần suất request trong cửa sổ 60 giây; cost guard kiểm soát tổng tiền đã dùng theo từng user trong tháng. Một user có thể gọi ít hơn 10 lần/phút nhưng đã gần hết ngân sách tháng, nên rate limit cho qua còn cost guard trả 402 trước khi gọi LLM. Ngược lại, user mới chưa tốn tiền nhưng bắn 11 request trong một phút: cost guard vẫn cho phép, rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng ping Redis, Redis mất kết nối sẽ làm cả ba container trả probe không khỏe. Orchestrator lần lượt coi từng container là lỗi, rút traffic rồi restart chúng. Trong 30 giây Redis chưa trở lại, các container vừa restart lại tiếp tục fail probe, làm cả cụm có thể không còn instance nhận request và tạo vòng restart không cần thiết. Tách `/health` giúp process vẫn sống, còn `/ready` trả 503 để load balancer tạm ngừng gửi request vào các instance đó.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, `history_length` tăng nhất quán theo cùng user (request đầu là 0, sau khi đã lưu một cặp message thì request sau là 2, rồi 4...). Redis là state dùng chung nên agent nào nhận request cũng đọc cùng một lịch sử. Nếu dùng dict Python, mỗi trong ba container có dict riêng; khi load balancer đổi container, con số có thể quay về 0 hoặc tăng theo một nhánh khác, khiến hội thoại mất ngữ cảnh.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lúc thử Railway, Redis báo `Build failed` và sau đó dashboard hiện `Payment method required` / workspace không thể tạo resource hoặc deploy. Tôi mở phần thông báo và nhận ra đây không phải lỗi Dockerfile hay Redis URL mà là hạn chế tài khoản Railway đã hết trial. Tôi không đưa secret lên GitHub và chuyển sang Render Blueprint: service `day12-agent` deploy thành công, Redis/Valkey khả dụng, rồi kiểm tra `/health` 200, `/ready` 200 với `redis: true`, và `/ask` thiếu key trả 401.
