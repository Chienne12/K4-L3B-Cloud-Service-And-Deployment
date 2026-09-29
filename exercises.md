# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu nằm dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phùng Trọng Chiến  Mã học viên: 2A202602430

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi tôi quên đặt `AGENT_API_KEY` trên Render, chương trình dừng ngay lúc khởi
động và báo thiếu biến môi trường. Nhờ vậy tôi biết cấu hình nào còn thiếu và
bổ sung secret trước khi public server. Nếu có mặc định `"changeme"`, app vẫn
chạy nhưng mọi người có thể đoán được key này và gọi API trái phép.
---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log tôi thu được:

```json
{"event":"ask_completed","level":"info","timestamp":"2026-09-29T07:38:07.347478+00:00","user_id":"sv-test","tokens_in":3,"tokens_out":8,"cost_usd":0.000011}
```

Từ dòng này tôi có thể lọc các request theo `event`, `user_id` hoặc thời gian;
đồng thời cộng `tokens` và `cost_usd` để theo dõi chi phí, tạo biểu đồ hay cảnh
báo. `print("đã trả lời xong")` không có các trường dữ liệu để làm hai việc đó.

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
| 1 stage (bản đầu) | 1.7 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Tôi đo bằng `docker image ls`: bản một stage dùng `python:3.11` là 1.7 GB,
còn bản multi-stage dùng `python:3.11-slim` là 271 MB. Phần chênh lệch chủ yếu
là hệ điều hành nền đầy đủ, công cụ build và các file trung gian không cần để
chạy app. Stage runtime chỉ nhận Python package đã cài và mã nguồn cần thiết.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa `app/main.py`, các layer base image, `COPY requirements.txt` và
`RUN pip install` được lấy từ cache. Docker chạy lại từ `COPY app ./app` và các
layer nằm sau nó. Nếu đặt `COPY . .` trước `RUN pip install`, mỗi lần sửa code
thì layer copy đổi, làm `pip install` chạy lại dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi rủi ro là: code Python có lỗ hổng → kẻ tấn công thực thi được lệnh trong
container → nếu process chạy root thì họ có quyền root trong container → nếu
container còn cấu hình nguy hiểm hoặc có lỗ hổng escape thì có thể tác động tới
host với quyền cao. `USER appuser` cắt chuỗi tại bước chiếm process: kẻ tấn
công chỉ nhận quyền của user thường, nên phạm vi phá hoại nhỏ hơn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request trong 2 giây: gửi 10 request ở giây 59
của phút trước, rồi gửi tiếp 10 request ở giây 00 của phút sau. Bộ đếm theo phút
đã reset dù 20 request thực tế nằm sát nhau. Sliding window 60 giây sẽ vẫn nhìn
thấy 10 request cũ và chặn nhóm tiếp theo.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tốc độ gọi trong một khoảng thời gian, còn cost guard giới
hạn tổng tiền đã dùng trong tháng. Một người gọi ít và đúng tốc độ nhưng đã gần
hết ngân sách thì rate limit cho qua, cost guard chặn. Ngược lại, một người gửi
quá nhiều request rất rẻ trong vài giây thì ngân sách vẫn còn nhưng rate limit
phải chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp hai endpoint, Redis mất kết nối → cả 3 container đều trả health check
lỗi → platform cho rằng cả 3 process đã chết → restart cả 3 container → Redis
vẫn chưa lên nên health check tiếp tục lỗi và cụm bị restart lặp lại. Tách riêng
thì `/health` vẫn báo process sống, còn `/ready` loại tạm container khỏi nhận
traffic cho đến khi Redis kết nối lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis dùng chung, các request dù rơi vào container nào vẫn đọc cùng lịch
sử nên `history_length` tăng đều `0, 2, 4, 6...` vì mỗi lượt lưu một câu hỏi và
một câu trả lời. Nếu dùng dict Python, mỗi container có lịch sử riêng; qua load
balancer tôi có thể thấy `0, 0, 0` rồi các số tăng không đều tùy request rơi vào
container nào. Dữ liệu cũng mất khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Sau khi deploy, tôi mở `/ask` trực tiếp trên trình duyệt và nhận
`405 Method Not Allowed`. Tôi kiểm tra route trong `app/main.py` và thấy endpoint
dùng `@app.post("/ask")`, trong khi trình duyệt đang gửi GET. Tôi sửa cách kiểm
tra: gửi POST bằng curl, kèm JSON body, `X-API-Key` và `X-User-Id`; endpoint sau
đó trả 200. Đây không phải lỗi server chết mà là gọi sai HTTP method.
