# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời ngay bên dưới từng câu hỏi.

> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Đức Thiện  Mã học viên: 2A202602032

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên môi trường Cloud/Production nhưng nhà phát triển quên cài đặt biến môi trường `AGENT_API_KEY`. Nếu để giá trị mặc định `"changeme"`, ứng dụng vẫn khởi động thành công và hoạt động bình thường, dẫn đến lỗ hổng bảo mật nghiêm trọng khi bất kỳ ai biết giá trị mặc định đều có thể truy cập API tự do, làm rò rỉ dữ liệu hoặc tiêu tốn chi phí token khổng lồ mà ta không hay biết. Ngược lại, việc không đặt mặc định khiến Pydantic ném `ValidationError` và làm app sập ngay lúc khởi động (Fail-fast). Điều này buộc nhà phát triển phải phát hiện và bổ sung ngay API Key trước khi service có thể nhận bất kỳ traffic nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:00:00.000000+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.0001}`

Hai việc làm được với log JSON:
1. **Truy vấn và lọc dữ liệu tự động**: Các công cụ quản lý log (Datadog, ElasticSearch, CloudWatch) có thể parse cấu trúc JSON để tìm kiếm chính xác theo trường (ví dụ: `level == "error"`, `user_id == "sv-test"` hoặc lọc các request có `cost_usd > 0.01`).
2. **Cảnh báo và vẽ biểu đồ theo định lượng**: Có thể tính tổng chi phí `cost_usd` theo thời gian thực hoặc thiết lập cảnh báo tự động khi số lượng event lỗi vượt ngưỡng trong vòng 5 phút, điều mà văn bản thuần tuý `print` không thể thực hiện tự động.

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
| 1 stage (bản đầu) | ~1.01 GB |
| Multi-stage | ~180 MB |

Giải thích: Phần dung lượng chênh lệch (~830 MB) là toàn bộ bộ biên dịch C/C++, build tools (gcc, make, python-dev), wheel cache, pip cache và các file tạm được sinh ra trong quá trình build dependency. Bản Multi-stage đã tách bạch giữa stage `builder` để biên dịch và stage `runner` chỉ copy duy nhất các package đã hoàn thiện sang runtime base image `python:3.11-slim` siêu nhẹ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile tối ưu: Layer `COPY requirements.txt` và `RUN pip install` được dùng lại hoàn toàn từ cache. Chỉ có layer `COPY . .` và các bước sau nó mới phải chạy lại.
Nếu đặt `COPY . .` lên trước `RUN pip install`: Khi sửa code trong `app/main.py`, layer `COPY . .` sẽ bị thay đổi làm vô hiệu hóa cache (cache invalidation) của tất cả các bước phía sau. Hệ thống buộc phải chạy lại lệnh `RUN pip install` từ đầu, gây lãng phí nhiều thời gian và băng thông mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python chứa lỗ hổng (ví dụ RCE / Arbitrary Code Execution).
2. Kẻ tấn công gửi payload khai thác lỗ hổng để thực thi lệnh shell bên trong container.
3. Vì container chạy với user `root` mặc định, tiến trình shell chiếm được có đặc quyền cao nhất (UID 0) bên trong container.
4. Kẻ tấn công thực hiện kỹ thuật Container Escape (khai thác lỗ hổng kernel hoặc cgroup/docker socket) để thoát khỏi container và chiếm luôn quyền `root` điều khiển toàn bộ máy host.

Lệnh `USER appuser` cắt đứt chuỗi ở bước 3: Tiến trình container bị giới hạn trong phạm vi của user thường (UID 1000), kể cả khi thực thi được lệnh độc hại cũng không có quyền can thiệp hệ thống hay thực hiện container escape lên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Số request tối đa: **20 request**.
Giải thích: Người dùng có thể gửi 10 request ở giây `10:00:59` (cuối phút thứ 1) và gửi tiếp 10 request ở giây `10:01:01` (đầu phút thứ 2 khi bộ đếm vừa reset về 0). Theo cơ chế Fixed Window, cả 2 khoảng thời gian đều hợp lệ (mỗi phút chỉ có 10 req), nhưng thực tế hệ thống bị quá tải với 20 request chỉ trong 2 giây liên tiếp. Thuật toán Sliding Window 60s giải quyết triệt để lỗ hổng này bằng cách tính tổng request trong đúng 60 giây trượt gần nhất.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Sự khác biệt: Rate Limit quản lý **tần suất/số lượng request** trong khoảng thời gian ngắn (ví dụ: max 10 req/phút). Cost Guard quản lý **tổng chi phí tài chính (USD)** phát sinh trong khoảng thời gian dài (ví dụ: max $10.0/tháng).
- Rate limit cho qua nhưng Cost guard chặn: User gửi 1 request duy nhất trong ngày (thỏa mãn rate limit), nhưng request đó kèm theo tài liệu cực kỳ dài làm chi phí LLM vượt ngân sách tháng còn lại.
- Cost guard cho qua nhưng Rate limit chặn: User mới bắt đầu tháng, ngân sách còn nguyên $10.0 (thỏa mãn cost guard), nhưng gửi dồn dập 50 request trong vòng 3 giây. Cost guard chưa cạn tiền nhưng Rate Limit lập tức chặn (429) để bảo vệ server khỏi nguy cơ sập do spam.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố mất kết nối trong 30 giây.
2. Endpoint gộp kiểm tra Redis bị lỗi/timeout.
3. Orchestrator (Docker/Kubernetes) nhận thấy Liveness probe thất bại nên kết luận cả 3 container ứng dụng đã bị hỏng/chết.
4. Orchestrator lập tức ra lệnh kill và khởi động lại (restart) cả 3 container.
5. Do Redis vẫn chưa khắc phục xong, các container mới khởi động lại tiếp tục fail health check và bị restart liên tục (CrashLoopBackOff). Cả hệ thống sập hoàn toàn thay vì chỉ tạm ngưng nhận request chờ Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict Python (Stateful): Mỗi request từ client sẽ được Load Balancer phân phối ngẫu nhiên đến 1 trong 3 instance container. Vì mỗi instance có bộ nhớ RAM riêng, con số `history_length` sẽ tăng giảm thất thường và bất nhất (ví dụ: req 1 vào node A -> len 0; req 2 vào node B -> len 0; req 3 vào node A -> len 2).
Nếu lưu ở Redis (Stateless): Cả 3 instance đều truy cập vào cùng một central Redis store, con số `history_length` sẽ tăng dần một cách nhất quán (0, 2, 4, 6...) bất kể request rơi vào node nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Thông báo lỗi: `redis.exceptions.ConnectionError: Error -2 connecting to localhost:6379. Name or service not known.`
Cách tìm nguyên nhân: Kiểm tra phần Runtime Logs trên Dashboard của dịch vụ Cloud (Railway/Render) sau khi deploy thất bại.
Cách sửa: Nguyên nhân do ở môi trường Cloud, Redis không còn chạy ở `localhost:6379`. Ta tạo một dịch vụ Redis Add-on trên Cloud, lấy đường dẫn kết nối của Cloud Redis đó và cấu hình lại biến môi trường `REDIS_URL` trong Dashboard của ứng dụng trên Cloud.
