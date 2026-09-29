# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bên dưới bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Tuấn Tú  Mã học viên: L3B202602840

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống cụ thể: Khi deploy ứng dụng lên môi trường production, lập trình viên hoặc DevOps quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard hoặc file `.env`.
> - Nếu có giá trị mặc định `"changeme"`: Container vẫn khởi động bình thường, health check báo xanh. Kẻ xấu hoặc bot tự động quét trên Internet sẽ thử các secret mặc định phổ biến (như `changeme`, `admin`, `secret`) và gọi API thành công. Chúng có thể spam hàng triệu request vào model LLM, làm rò rỉ dữ liệu hoặc đốt cháy hàng ngàn USD tiền API của bạn trước khi bạn kịp phát hiện.
> - Ngược lại khi không có mặc định (Fail Fast): Pydantic ném `ValidationError` ngay lúc tiến trình nạp cấu hình và container dừng ngay lập tức khi vừa deploy. Lỗi hiện rõ trên deployment log giúp dev phát hiện và bổ sung secret ngay trước khi service mở ra ngoài internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:30:15.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.00015}`
>
> Hai việc làm được với log JSON mà `print()` không làm được:
> 1. **Lọc, truy vấn và tổng hợp định lượng (Aggregation & Metrics):** Hệ thống gom log tập trung (Datadog, Loki, CloudWatch, ELK) có thể phân tích cú pháp (parse) các trường JSON trực tiếp để chạy query như: *"Tổng chi phí `cost_usd` của user `sv-test` trong 24 giờ qua là bao nhiêu?"*, hoặc *"Vẽ biểu đồ số lượng token (`tokens_in`, `tokens_out`) tiêu thụ theo thời gian"*. Chuỗi `print("đã trả lời xong")` chỉ là text thuần vô cấu trúc, không thể tính toán hay filter theo trường dữ liệu.
> 2. **Tự động kích hoạt cảnh báo (Automated Alerting):** Có thể đặt rule cảnh báo tự động khi phát hiện các sự kiện bất thường, ví dụ: cảnh báo về Slack/PagerDuty khi `level == "error"` vượt quá 5% số request trong 5 phút, hoặc khi `cost_usd > 0.05` trên một request đơn lẻ.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~750 MB) bao gồm:
> 1. Base image đầy đủ (`python:3.11`) chứa cả hệ điều hành Debian đầy đủ kèm theo trình biên dịch C/C++ (`gcc`, `g++`, `make`), các thư viện phát triển (`linux-headers`, `libssl-dev`, `python3-dev`) cần để biên dịch thư viện C-extensions.
> 2. Stage builder chỉ dùng các công cụ biên dịch này để cài đặt thư viện vào thư mục đích (`/install`). Ở stage runtime, chúng ta dùng `python:3.11-slim` và chỉ `COPY --from=builder /install /usr/local`. Toàn bộ trình biên dịch, file header, công cụ build và cache tạm thời của pip đều bị loại bỏ lại ở stage 1, không lọt vào image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Với Dockerfile hiện tại:
>   - Các layer trước `COPY app ./app` (bao gồm: tải base image, tạo workdir, `COPY requirements.txt`, `RUN pip install`, tạo `appuser`) đều được dùng lại hoàn toàn từ cache (`CACHED`).
>   - Chỉ có layer `COPY --chown=appuser:appuser app ./app` và các chỉ thị tiếp theo (`COPY utils`, `HEALTHCHECK`, `CMD`) là phải chạy lại. Quá trình build lại chỉ mất 1-2 giây.
> - Nếu đặt `COPY . .` lên trước `RUN pip install`:
>   - Mỗi khi sửa dù chỉ một ký tự trong `app/main.py`, layer `COPY . .` sẽ bị invalid cache (vì checksum thư mục thay đổi).
>   - Do Docker huỷ toàn bộ cache từ layer thay đổi trở đi, lệnh `RUN pip install` phía sau sẽ bị ép chạy lại từ đầu, tải và cài đặt lại toàn bộ thư viện mỗi lần sửa code, làm quá trình build mất vài phút thay vì vài giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện tấn công (Container Escape):
> 1. Kẻ tấn công phát hiện một lỗ hổng trong code Python (ví dụ: Remote Code Execution qua `eval()`, deserialization không an toàn qua `pickle`, hoặc lỗ hổng command injection).
> 2. Kẻ tấn công thực thi được mã tùy ý trong tiến trình Python. Vì container chạy mặc định bằng user `root` (UID 0), tiến trình Python có đầy đủ quyền root bên trong container.
> 3. Từ quyền root trong container, kẻ tấn công khai thác các lỗ hổng nhân Linux (kernel exploits như Dirty COW/Dirty Pipe), hoặc lợi dụng các cấu hình mount không an toàn (như mount `/var/run/docker.sock` hoặc thư mục `/host`), để thoát khỏi cgroups/namespaces (Container Escape).
> 4. Vì UID 0 trong container mặc định ánh xạ trùng với UID 0 (root) trên máy host, kẻ tấn công chiếm toàn quyền kiểm soát cao nhất (root) trên chính máy chủ host vật lý.
>
> Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: Tiến trình Python chạy dưới quyền user thường không có đặc quyền (`UID 10001`). Khi bị RCE, kẻ tấn công chỉ có quyền của user thường trong container, không thể ghi vào thư mục hệ thống, không có quyền can thiệp vào kernel interface hay tận dụng các capability đặc quyền để escape sang host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> - Số request tối đa trong 2 giây liên tiếp: **20 request**.
> - Giải thích:
>   - Với cách đếm theo phút đồng hồ (Fixed Window), bộ đếm sẽ reset về 0 tại mốc giây 00 của mỗi phút (ví dụ 10:00:00, 10:01:00).
>   - Người dùng có thể gửi 10 request vào giây cuối cùng của phút thứ nhất: **10:00:59** (10 request này được tính vào phút 10:00, quota còn lại = 0).
>   - Đúng 1 giây sau, đồng hồ chuyển sang **10:01:00** (bước sang phút mới), bộ đếm tự động reset về 0. Người dùng lập tức gửi tiếp 10 request tại giây **10:01:00**.
>   - Kết quả: Trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:00), hệ thống đã nhận tới 20 request (gấp đôi hạn mức quy định), gây nguy cơ quá tải dịch vụ.
>   - Cửa sổ trượt (Sliding Window) giải quyết triệt để vấn đề này vì nó luôn tính tổng request trong khoảng `[now - 60s, now]`, nên tại bất kỳ thời điểm nào người dùng cũng không thể gửi quá 10 request trong 60 giây liên tiếp.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - Điểm khác nhau:
>   - **Rate Limit:** Giới hạn theo **tần suất / số lượng request trong một khoảng thời gian ngắn** (ví dụ 10 req/phút). Mục tiêu là chống tấn công DoS/spam và bảo vệ hạ tầng máy chủ khỏi bị quá tải.
>   - **Cost Guard:** Giới hạn theo **tổng chi phí tài chính (USD) trong một khoảng thời gian dài** (ví dụ 10 USD/tháng). Mục tiêu là kiểm soát ngân sách, ngăn chặn hóa đơn API bên thứ ba phình to bất ngờ.
> - Hai tình huống:
>   1. *Rate limit cho qua nhưng Cost guard chặn:* Người dùng chỉ gửi 1 request trong cả ngày (hoàn toàn dưới 10 req/phút), nhưng câu hỏi đó quá dài (hoặc yêu cầu tạo prompt 100.000 tokens) khiến chi phí vượt quá ngân sách tháng còn lại của người dùng. Cost guard sẽ chặn ngay với mã lỗi 402, trong khi Rate limit vẫn cho qua.
>   2. *Cost guard cho qua nhưng Rate limit chặn:* Người dùng mới đăng ký, ngân sách còn nguyên 10 USD chưa tiêu đồng nào, nhưng lại gửi 15 request "test" ngắn liên tiếp trong vòng 5 giây. Cost guard thấy tài khoản còn đủ tiền nên cho qua, nhưng Rate limit sẽ chặn từ request thứ 11 với mã lỗi 429 vì tốc độ gọi quá nhanh.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện (Cascading Failure):
> 1. **Giây 0:** Redis gặp sự cố mạng hoặc bị restart tạm thời trong 30 giây.
> 2. **Giây 1 - 5:** Orchestrator (Docker/Kubernetes) thực hiện định kỳ gọi liveness probe (`/health`) vào cả 3 container agent. Vì endpoint này phụ thuộc vào Redis, kết nối ping Redis thất bại nên cả 3 container đồng loạt trả về lỗi 503 hoặc timeout.
> 3. **Giây 6 - 15:** Orchestrator thấy liveness probe thất bại vượt ngưỡng số lần thử (retries) nên kết luận rằng tiến trình container đã chết (unhealthy). Orchestrator gửi lệnh `SIGKILL` để restart toàn bộ 3 container cùng một lúc.
> 4. **Giây 15 - 25:** Cả 3 container đang trong quá trình khởi động lại hoặc CrashLoopBackOff. Trong thời gian này, không còn bất kỳ container nào còn sống để nhận traffic, toàn bộ request của người dùng đều nhận lỗi 502/503.
> 5. **Giây 30:** Khi Redis hoạt động trở lại bình thường, các container vẫn đang chật vật khởi động lại hoặc bị kẹt chu kỳ restart.
> -> *Kết luận:* Một sự cố tạm thời của dependency bên ngoài đã biến thành sự cố sập toàn bộ hệ thống (Total Outage) chỉ vì liveness probe phụ thuộc vào bên ngoài. Đúng chuẩn: `/health` chỉ kiểm tra tiến trình app còn sống không; còn `/ready` mới kiểm tra dependency để load balancer tạm thời ngừng đẩy traffic chứ không restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - Với Redis (Stateless): `history_length` tăng đều đặn theo mỗi lượt hỏi (0 ➔ 2 ➔ 4 ➔ 6...) vì cả 3 container đều đọc và ghi chung một nguồn dữ liệu lịch sử trong Redis.
> - Nếu lưu trong một dict Python trong RAM (Stateful):
>   - Khi load balancer điều hướng request theo thuật toán Round-Robin hoặc ngẫu nhiên sang các container A, B, C:
>   - Lượt 1 vào container A: A ghi vào RAM của nó, trả `history_length = 0`.
>   - Lượt 2 vào container B: RAM của B đang trống rỗng, B cũng trả `history_length = 0` (thay vì 2)!
>   - Lượt 3 vào container C: RAM của C cũng trống, trả `history_length = 0`.
>   - Lượt 4 quay lại container A: A thấy 2 tin nhắn cũ của lượt 1, trả `history_length = 2`.
>   - Kết quả: `history_length` sẽ nhảy lung tung (0, 0, 0, 2, 0, 2...) tùy thuộc vào request rơi vào container nào. Agent sẽ bị "mất trí nhớ ngẫu nhiên" giữa các lượt hội thoại vì state bị phân mảnh trong RAM riêng của từng process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Tình huống lỗi: Khi cấu hình deploy dịch vụ lên nền tảng đám mây (Render) theo hướng dẫn Blueprint `render.yaml`, vừa tạo tài khoản và trigger deploy thì nhận được thông báo lỗi từ hệ thống:
>   `"Your account has been suspended for suspicious activity. If you believe this was a mistake, please contact support."`
> - Cách tìm ra nguyên nhân:
>   - Xem thông báo trên giao diện và tra cứu tài liệu hỗ trợ của Render: Hệ thống kiểm duyệt tự động (Anti-Abuse AI) của Render quét gắt gao các tài khoản mới tạo và tự động gắn cờ đình chỉ đối với các tài khoản dùng IP mạng công cộng/VPN hoặc các template có dịch vụ Redis/Key-Value vì nghi ngờ đào coin.
> - Cách xử lý:
>   - Chuyển hướng hạ tầng sang giải pháp tự chủ: Tận dụng server cá nhân đã có sẵn Docker và Nginx Proxy Manager, kết hợp cấu hình reverse proxy trỏ domain `lab12.tutran-dev.id.vn`. Đóng gói Docker image chuẩn multi-stage đẩy lên Docker Hub và kéo về server chạy độc lập. Đồng thời kích hoạt phương án dự phòng chuẩn `LOCAL_FALLBACK=true` của bài lab để kiểm thử và đối soát kết quả ổn định.

