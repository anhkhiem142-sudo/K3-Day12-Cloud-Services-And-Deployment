# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết nội dung của bạn ngay dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Minh Khiêm  Mã học viên: 2A202601645

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu tôi quên đặt `AGENT_API_KEY` thì `Settings` báo lỗi ngay và container không nhận traffic. Nhờ vậy tôi biết dashboard còn thiếu secret trước khi public API. Nếu dùng mặc định `changeme`, service vẫn báo khỏe nhưng người khác có thể đoán khóa này và gọi `/ask`, làm tiêu quota mà tôi khó phát hiện sớm.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi thu được là `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T04:39:45.896590+00:00","user_id":"cp4-scale-check","tokens_in":195,"tokens_out":48,"cost_usd":5.805e-05}`. Với JSON này tôi có thể lọc toàn bộ request của một `user_id`, và tổng hợp token/chi phí hoặc cảnh báo theo trường số. Dòng `print("đã trả lời xong")` không có các trường ổn định để máy tự parse và thống kê như vậy.

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
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại Dockerfile ban đầu từ Git history và đo được bản một stage là 1.73 GB, còn image production multi-stage là 270 MB. Phần chênh lệch chủ yếu đến từ base image Python đầy đủ, công cụ/hệ thống phục vụ build và các nội dung source/cache không cần thiết ở runtime. Stage cuối chỉ dùng `python:3.11-slim`, dependencies đã cài và hai thư mục `app`, `utils`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `WORKDIR`, `COPY requirements.txt`, `pip install` và copy dependencies từ builder được dùng lại từ cache. Docker chỉ chạy lại từ `COPY app/ app/` trở về sau. Nếu `COPY . .` đứng trước `RUN pip install`, một thay đổi source làm layer copy đổi, kéo theo việc cài lại toàn bộ dependencies dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng cho phép chạy lệnh, kẻ tấn công sẽ chiếm quyền của process trong container. Khi process chạy root, họ có quyền cao nhất trong container và có thêm cơ hội khai thác cấu hình/mount hoặc lỗ hổng runtime để ảnh hưởng host. `USER appuser` cắt chuỗi ở bước chiếm process: mã bị khai thác chỉ nhận quyền hạn chế của `appuser`, không mặc nhiên là root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong khoảng hai giây quanh ranh giới phút: gửi 10 request ở giây 59 của phút trước, rồi gửi thêm 10 request ở giây 00 của phút sau. Fixed window coi đó là hai cửa sổ khác nhau. Sliding window 60 giây vẫn nhìn thấy cả nhóm cũ khi nhóm mới đến nên chặn khi tổng số trong 60 giây đạt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit bảo vệ tốc độ và tải hạ tầng; cost guard bảo vệ số tiền tích lũy. Một user gửi một request mỗi phút với tài liệu rất dài sẽ không quá nhanh nhưng có thể bị cost guard chặn vì tốn nhiều token. Ngược lại, user gửi nhiều câu cực ngắn trong vài giây có thể còn rất xa ngân sách nhưng bị rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint liveness cũng ping Redis, khi Redis mất kết nối thì cả ba container cùng báo lỗi health. Orchestrator hiểu nhầm cả ba process đã chết, lần lượt restart chúng; container mới vẫn không kết nối được Redis nên tiếp tục fail và tạo vòng lặp restart. Các request đang xử lý bị gián đoạn dù code agent vẫn sống. Tách `/health` giúp process không bị restart oan, còn `/ready` trả 503 để load balancer tạm ngừng gửi traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với ba replica và Redis chung, năm lần gọi thực tế cho `history_length` lần lượt là 0, 2, 4, 6, 8 vì mỗi lượt lưu một message user và một message assistant. Nếu dùng dict Python, mỗi container có lịch sử riêng; Nginx chuyển request qua các replica sẽ làm số này quay về 0 hoặc tăng rời rạc theo lịch sử riêng của từng container, thay vì tăng đều trên toàn hệ thống.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy Railway đầu tiên fail với lỗi `Invalid value for '--port': '$PORT' is not a valid integer`. Tôi xem deployment logs và thấy Uvicorn nhận nguyên chuỗi `$PORT`, chứng tỏ `startCommand` không chạy qua shell để expand biến. Tôi sửa `railway.toml` thành `sh -c 'exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'`, push commit và redeploy từ GitHub; deployment sau đó chuyển sang `SUCCESS` và cả `/health`, `/ready`, `/ask` đều trả status mong đợi.
