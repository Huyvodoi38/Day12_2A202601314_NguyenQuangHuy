# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Quang Huy  Mã học viên: 2A202601314

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên cloud, nếu quên set biến môi trường `API_TOKEN` và app dùng mặc định `"changeme"`, ứng dụng vẫn khởi động thành công. Kẻ xấu hoặc bot quét internet có thể dò thấy endpoint công khai và sử dụng dịch vụ LLM bằng token mặc định đó mà bạn không hề hay biết, dẫn đến hóa đơn LLM tăng vọt. Nếu không có giá trị mặc định, app sẽ "chết ngay" (`ValidationError`) ngay lúc vừa deploy, buộc developer phải vào cấu hình đúng secret trước khi app có thể nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON mẫu thu được:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:05:00.123456+00:00", "client_id": "sv-test", "prompt_tokens": 15, "completion_tokens": 40, "usd_cost": 0.00012}`

Hai việc log JSON làm được mà `print()` truyền thống không làm được:

1. **Lọc và tìm kiếm theo thuộc tính có cấu trúc (Structured Query)**: Các hệ thống quản lý log (Cloud Logging, Datadog) có thể dễ dàng lọc theo `severity == "ERROR"` hoặc `client_id == "sv-test"`.
2. **Thống kê & Cảnh báo tự động (Aggregation & Alerting)**: Có thể tự động tính tổng chi phí `usd_cost` theo ngày của từng client hoặc thiết lập cảnh báo khi tỷ lệ lỗi 5xx tăng bất thường.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản                 | Dung lượng |
| -------------------- | ------------ |
| 1 stage (bản đầu) | ~1.8 GB      |
| Multi-stage          | ~270 MB      |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~1.5 GB) bao gồm các công cụ biên dịch (gcc, g++, make, build-essential), bộ header C, trình quản lý gói đầy đủ và bộ nhớ đệm pip/apt của stage builder. Stage runtime chỉ copy kết quả biên dịch và dùng base image `python:3.11-slim` nhỏ gọn, loại bỏ toàn bộ công cụ build không cần thiết khi ứng dụng thực thi.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa một ký tự trong `app/main.py`:

- Các layer `FROM`, `WORKDIR`, `COPY requirements.txt`, `RUN pip install` và `COPY --from=builder` được dùng lại từ Docker cache.
- Chỉ có layer `COPY . .` và các lệnh phía sau (`RUN useradd`, `USER`, `HEALTHCHECK`, `CMD`) mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa một ký tự trong code, layer `COPY . .` bị hủy cache, kéo theo lệnh `RUN pip install` phải tải và cài lại toàn bộ thư viện từ đầu, khiến thời gian build tăng từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:

1. Ứng dụng Python bị dính lỗ hổng thực thi lệnh từ xa (Remote Code Execution - RCE).
2. Kẻ tấn công gửi payload khai thác lỗ hổng để chạy lệnh shell bên trong container.
3. Nếu container chạy dưới quyền `root`, kẻ tấn công có quyền root trong container, có thể truy cập toàn bộ hệ thống file container và tìm cách leo thang quyền qua các volume/mount point để chiếm quyền root trên máy host.
   Lệnh `USER appuser` cắt đứt chuỗi ở bước 3: Process chạy dưới quyền user thường (UID 10001), không có quyền root nên không thể đọc/ghi các file hệ thống quan trọng hay thực hiện các thao tác nguy hiểm trên máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- 401 bắt buộc phải kèm header `WWW-Authenticate: Bearer` theo chuẩn HTTP (RFC 6750) để thông báo cho client biết phương thức xác thực mà API yêu cầu.
- Ta trả cùng một thông báo lỗi `"invalid or missing bearer token"` cho cả 3 trường hợp để tránh rò rỉ thông tin (information disclosure). Nếu báo rõ "sai scheme" hay "sai token", kẻ tấn công sẽ biết chúng đã đi đúng bước nào và tập trung dò tìm/brute-force token.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- Với `capacity=10`, `refill_per_minute=10`: Sau 10 phút im lặng, xô cũng chỉ nạp đầy tối đa `capacity = 10` token. Do đó, client gửi liên tiếp được **10 request** trước khi bị trả lỗi 429.
- Nếu bỏ đoạn `min(capacity, ...)`: Sau 10 phút (600 giây), lượng token nạp dồn theo công thức là `10 * 10/60 * 60 = 100` token. Client sẽ gửi bùng nổ được **110 request** liên tiếp trong 1 giây, làm vô hiệu hóa cơ chế rate limit và gây quá tải hệ thống.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- **Với hạn mức $30/tháng**: Sự cố lúc 2h sáng có thể thiêu rụi toàn bộ $30 ngân sách của cả tháng chỉ trong vài giờ. Service sẽ bị khóa cho đến đầu tháng sau (khoảng 20-30 ngày).
- **Với hạn mức $1/ngày**: Thiệt hại tối đa của sự cố bị giới hạn ở **$1.0**. Khi vượt quá $1, API chặn lại (402). Đến 00:00 UTC sáng hôm sau, nhãn ngày đổi sang ngày mới, service **tự động phục hồi** mà không cần con người can thiệp.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:

1. Redis mất kết nối trong 30 giây.
2. Endpoint gộp trả về 503 cho cả 3 container.
3. Orchestrator (Docker / Kubernetes) hiểu nhầm rằng các container đã bị sập ➔ tiến hành **restart đồng loạt cả 3 container**.
4. Cả 3 container bị restart cùng lúc trong khi Redis vẫn chưa hồi phục ➔ rơi vào vòng lặp khởi động lại liên tục.
5. Khi Redis hoạt động trở lại, cả 3 container đều đang dở dang quá trình restart ➔ không có container nào sẵn sàng phục vụ người dùng, biến một sự cố nhỏ của Redis thành sự cố sập toàn bộ hệ thống.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**: `400 Bad Request` ("There was an error parsing the body") và `401 Unauthorized` khi test endpoint `/chat` từ terminal Git Bash trên Windows.
- **Cách tìm ra nguyên nhân**: Đọc log phản hồi của cURL và kiểm tra lại môi trường shell. Phát hiện ra biến `$API_TOKEN` chưa được `export` vào phiên làm việc của Git Bash nên cURL gửi header token rỗng. Đồng thời cURL trong Git Bash Windows gặp lỗi mã hóa ký tự Unicode có dấu trong body JSON.
- **Cách sửa**: Gán `export API_TOKEN="..."` hoặc điền trực tiếp giá trị token vào header cURL, đồng thời truyền chuỗi JSON không dấu (`{"message": "Deploy la gi?"}`).
