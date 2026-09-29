# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Quang Đạo  Mã học viên: 2A202602394

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên môi trường Cloud (như Railway/Render/Kubernetes), người deploy vô tình quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard.
- Nếu để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường, healthcheck báo 200 OK, hệ thống cho phép nhận traffic từ bên ngoài. Lúc này, bất kỳ bot quét Internet hoặc kẻ xấu nào cũng có thể thử các API key mặc định phổ biến như `"changeme"` để gọi API, làm rò rỉ dữ liệu hoặc đốt sạch ngân sách/token của tài khoản LLM trước khi người quản trị nhận ra.
- Khi không có giá trị mặc định, Pydantic sẽ ném ngay lỗi `ValidationError` lúc khởi động (fail-fast), container bị crash ngay lập tức trong quá trình deploy. Người triển khai thấy lỗi hiển thị ngay trên màn hình dashboard và bắt buộc phải bổ sung secret hợp lệ trước khi mở traffic cho công chúng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

**Dòng log JSON thu được:**
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:15:38.123456+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

**Hai việc làm được với log có cấu trúc JSON:**
1. **Truy vấn, lọc và tổng hợp chi phí theo người dùng (Aggregation & Analytics):** Do log có các trường khóa rõ ràng (`user_id`, `cost_usd`, `tokens_in`, `tokens_out`), các công cụ quản lý log (như Datadog, Grafana Loki, CloudWatch, ELK) có thể dễ dàng phân tích và trả lời các câu hỏi kinh doanh như: *"Hôm nay user nào tiêu nhiều token nhất?"*, *"Tổng chi phí LLM trong 1 giờ qua là bao nhiêu?"*. Câu lệnh `print()` dạng văn bản thuần đòi hỏi phải viết regex phức tạp và dễ vỡ khi thay đổi câu chữ.
2. **Thiết lập cảnh báo tự động theo ngưỡng (Automated Alerting):** Có thể cấu hình cảnh báo ngay lập tức (qua Slack/PagerDuty) khi trường `cost_usd` của một request vượt quá ngưỡng bất thường hoặc khi tỷ lệ lỗi (dựa trên trường `level`: "error") trong 5 phút vượt quá 5%, giúp phát hiện sớm sự cố hoặc hành vi lạm dụng.

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
| Multi-stage | ~175 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~845 MB) gồm:
1. **Công cụ build và biên dịch của hệ điều hành:** Bản 1-stage dùng base image `python:3.11` đầy đủ, đi kèm toàn bộ bộ công cụ Debian build essentials (GCC, G++, Make), các file header C/C++ và các gói phát triển hệ thống. Bản multi-stage dùng base image `python:3.11-slim` chỉ giữ lại kernel runtime tối thiểu.
2. **Tách biệt môi trường build và runtime:** Ở bản multi-stage, các công cụ biên dịch và bộ nhớ đệm cài đặt của pip chỉ nằm ở stage `builder`. Stage `runtime` cuối cùng chỉ copy kết quả các package đã cài đặt từ `/install` sang `/usr/local`, hoàn toàn không mang theo compiler, tài liệu man pages, apt cache hay pip cache (`--no-cache-dir`).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Với Dockerfile hiện tại:**
  - Các layer trước đó: `COPY requirements.txt .`, `RUN pip install --no-cache-dir ...`, tạo user `appuser`, và `COPY --from=builder /install /usr/local` đều được **dùng lại hoàn toàn từ cache** (`CACHED`).
  - Chỉ các layer từ `COPY app ./app` trở đi mới bị tính lại checksum và phải chạy lại. Do đó quá trình build lại chỉ mất 1 - 2 giây.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Mỗi khi sửa dù chỉ một ký tự trong code (`app/main.py`), checksum của layer `COPY . .` sẽ thay đổi.
  - Theo cơ chế caching của Docker, một khi một layer bị thay đổi thì toàn bộ các layer tiếp theo phía sau đều bị hủy cache (cache invalidated).
  - Hậu quả: Docker buộc phải chạy lại lệnh `RUN pip install` từ đầu, tải lại và cài đặt lại toàn bộ dependencies trong `requirements.txt`, làm thời gian build kéo dài thêm vài phút mỗi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

**Chuỗi sự kiện leo thang đặc quyền:**
1. Ứng dụng Python tồn tại một lỗ hổng bảo mật (ví dụ: Remote Code Execution qua deserialization không an toàn, command injection, hoặc thư viện bên thứ ba có lỗ hổng zero-day).
2. Kẻ tấn công gửi payload khai thác lỗ hổng và mở được một reverse shell hoặc thực thi lệnh shell bên trong container. Vì container mặc định chạy bằng `root`, process của kẻ tấn công có UID 0 (root bên trong container).
3. Kẻ tấn công thực hiện kỹ thuật container breakout / escape (ví dụ: khai thác lỗ hổng nhân Linux của máy host, hoặc lợi dụng container được mount thư mục nhạy cảm hay Docker socket `/var/run/docker.sock`).
4. Vì process chạy bằng UID 0, khi thoát được ra ngoài host, nó tương ứng trực tiếp với UID 0 (root) của hệ điều hành host $\to$ kẻ tấn công chiếm toàn quyền điều khiển máy chủ host vật lý/máy ảo.

**Lệnh `USER appuser` cắt đứt chuỗi ở đâu:**
Lệnh `USER appuser` cắt đứt chuỗi ngay tại **bước 2**: Process bên trong container bị hạ quyền xuống UID 10001 (non-root). Khi kẻ tấn công chiếm được shell, chúng chỉ có quyền của user thường: không thể ghi vào các file hệ thống, không thể load kernel module, không thể can thiệp vào các tiến trình khác hoặc mở các raw socket đặc quyền. Điều này làm vô hiệu hóa hầu hết các phương thức khai thác container breakout.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

**Con số tối đa:** **20 request** trong 2 giây liên tiếp.

**Giải thích:**
- Với thuật toán đếm theo phút đồng hồ (Fixed Window Counter), bộ đếm sẽ tự động reset về 0 vào đầu mỗi phút (tại giây :00).
- Kẻ tấn công hoặc người dùng có thể căn thời gian gửi 10 request vào giây cuối cùng của phút thứ nhất: từ `10:00:59`. Vì 10 request này nằm trong khung giờ `10:00:xx` nên hệ thống cho qua.
- Ngay sau đó đúng 1 giây, đồng hồ điểm `10:01:00`, bộ đếm được reset về 0. Người dùng gửi tiếp ngay 10 request nữa vào giây `10:01:01`. Vì 10 request này nằm trong khung giờ `10:01:xx` nên hệ thống tiếp tục cho qua.
- Kết quả: Người dùng đã gửi tổng cộng 20 request trong vòng 2 giây (từ 10:00:59 đến 10:01:01) mà không vi phạm luật đếm theo phút, gây đột biến tải tức thời gấp đôi công suất cho phép. Thuật toán Sliding Window (cửa sổ trượt) giải quyết triệt để lỗi này bằng cách luôn tính toán chính xác số request trong bất kỳ khoảng thời gian 60 giây liên tục nào.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

**Điểm khác nhau cốt lõi:**
- **Rate Limit**: Giới hạn **tần suất / số lượng request** trong một khoảng thời gian ngắn (ví dụ: 10 request / phút) để bảo vệ hạ tầng máy chủ khỏi bị nghẽn mạng, quá tải CPU và tấn công từ chối dịch vụ (DoS).
- **Cost Guard**: Giới hạn **tổng chi phí / ngân sách tài chính tích lũy** (USD hoặc tổng token) trong một chu kỳ dài (ví dụ: 10.0 USD / tháng) để bảo vệ ví tiền của chủ sở hữu hệ thống khỏi việc hóa đơn LLM tăng đột biến.

**Tình huống Rate limit cho qua nhưng Cost guard phải chặn:**
Một người dùng chỉ gửi 2 request trong 1 phút (hoàn toàn dưới ngưỡng rate limit 10 request/phút), nhưng mỗi request gửi kèm một tài liệu khổng lồ 50,000 token khiến chi phí mỗi lượt gọi là $0.50. Sau nhiều request trong tháng, người dùng này đã chạm mốc ngân sách $10.0. Ở request tiếp theo, Rate limit thấy tần suất thấp nên cho qua, nhưng Cost guard kiểm tra thấy ngân sách tháng đã cạn và chặn lại ngay lập tức với mã lỗi `402 Payment Required`.

**Tình huống Cost guard cho qua nhưng Rate limit phải chặn:**
Đầu tháng, một người dùng mới chỉ tiêu $0.02 trong hạn mức $10.0. Người dùng viết một đoạn mã vòng lặp gửi liên tục 15 request câu hỏi ngắn (mỗi câu chỉ tốn $0.0001) trong vòng 5 giây. Về mặt ngân sách, người dùng còn dư tới $9.98 (Cost guard cho qua), nhưng tần suất gọi quá nhanh vi phạm hạn mức 10 req/phút nên Rate limit chặn lại ngay từ request thứ 11 với mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

**Thứ tự sự kiện xảy ra:**
1. Redis gặp sự cố mạng hoặc khởi động lại, mất kết nối trong 30 giây.
2. Endpoint kiểm tra sức khỏe của cả 3 container agent đồng loạt gọi vào Redis và bị fail/timeout.
3. Vì endpoint này đóng vai trò Liveness probe (kiểm tra sự sống của process), orchestrator (Docker/Kubernetes/Cloud) coi rằng cả 3 container agent đều đã bị treo/chết.
4. Orchestrator lập tức kill và restart lại toàn bộ cả 3 container agent cùng lúc để cố gắng tự phục hồi.
5. Toàn bộ hệ thống agent rơi vào tình trạng sập hoàn toàn (100% outage), các request người dùng đang xử lý dở bị đứt gãy và trả về lỗi `502 Bad Gateway`.
6. Khi 3 container mới khởi động lên, Redis vẫn chưa hoàn tất 30 giây phục hồi $\to$ các container tiếp tục kiểm tra fail $\to$ orchestrator lại tiếp tục restart liên tục (CrashLoopBackOff).
7. Khi Redis kết nối trở lại, cụm container vẫn chưa sẵn sàng ngay mà phải mất thêm thời gian khởi động lại từ đầu. Sự cố tạm thời ở tầng phụ thuộc (cache) đã biến thành thảm họa sập toàn bộ dịch vụ. (Tách `/ready` riêng sẽ giúp load balancer chỉ tạm ngưng đẩy traffic vào mà không restart container).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Khi lưu trên Redis (Stateless):**
  `history_length` tăng đều đặn và liên tục qua từng lượt hỏi: `0 -> 2 -> 4 -> 6 -> 8...` bất kể request được load balancer đẩy ngẫu nhiên vào container A, container B hay container C, vì cả 3 container đều cùng đọc/ghi chung một cơ sở dữ liệu Redis.
- **Nếu lưu trong dict Python trong RAM của process (Stateful):**
  - Mỗi container chỉ giữ bộ nhớ riêng trong RAM của chính nó.
  - Khi load balancer phân phối xoay vòng (round-robin) các request:
    - Lượt 1 vào container A: Container A ghi nhớ 2 message (`history_length` ban đầu = 0).
    - Lượt 2 vào container B: Container B chưa thấy lượt 1 bao giờ, nên response báo `history_length = 0`!
    - Lượt 3 vào container C: Container C cũng chưa thấy lượt nào, response tiếp tục báo `history_length = 0`.
    - Lượt 4 quay lại container A: Lúc này mới thấy lại lịch sử của lượt 1, response báo `history_length = 2`.
  - Kết quả là `history_length` nhảy hỗn loạn, agent bị hiện tượng "mất trí nhớ ngẫu nhiên" và không thể duy trì ngữ cảnh hội thoại mạch lạc với người dùng.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Lỗi kết nối Redis trên Railway khiến endpoint `/ready` trả về mã lỗi 503 dù ứng dụng đã deploy thành công và `/health` đã báo 200 OK.
- **Thông báo lỗi nhận được:**
  ```http
  HTTP/1.1 503 Service Unavailable
  {"status":"not ready","redis":false}
  ```
- **Cách tìm ra nguyên nhân:**
  - Nhận thấy `/health` trả về 200 chứng minh process Python và Uvicorn đang chạy bình thường, nhưng `/ready` kiểm tra phụ thuộc `store.ping()` lại thất bại.
  - Kiểm tra tab Variables của service `day12-agent` trên Railway dashboard, phát hiện biến `REDIS_URL` chưa được liên kết với service Redis nội bộ mà vẫn đang nhận giá trị mặc định `redis://localhost:6379/0`. Trong môi trường container của Railway, `localhost` trỏ về chính container của agent chứ không phải container Redis.
- **Cách sửa:**
  - Trên Railway dashboard, vào service `day12-agent` $\to$ tab **Variables** $\to$ sửa biến `REDIS_URL` bằng cách nhấn **Add Reference** và chọn biến `${{Redis.REDIS_URL}}` của service Redis.
  - Chờ Railway tự động redeploy lại bản build mới, sau đó kiểm tra lại bằng lệnh `curl.exe -i https://day12-agent-production-9ba1.up.railway.app/ready` $\to$ nhận về kết quả thành công `200 OK {"status":"ready","redis":true}`.
