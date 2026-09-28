# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Đức Tùng  Mã học viên: 2A202603005
---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Thay vì silence failure thì loudly failure, lúc đó lỗi dần người dev devops nhanh hơn để nó lên đến môi trường khác thì cũng trở nên nghiêm trọng

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log thu được:
>
> {
  "answer": "Theo mình hiểu, string liên quan tới cách hệ thống được đóng gói và vận hành. Điểm mấu chốt là tách cấu hình ra khỏi code và giữ service ở trạng thái stateless.",
  "user_id": "Toi la T, toi muon thanh expert",
  "history_length": 0,
  "cost_usd": 0.00002415,
  "tokens": {
    "in": 1,
    "out": 40
  }
}
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. Lọc và tổng hợp theo trường: vì có `user_id`, `cost_usd`, `tokens_in`, tôi
>    dùng `jq` hoặc công cụ log để tính "user nào đã tốn bao nhiêu tiền hôm nay"
>    hoặc chỉ lấy các dòng `level=error`. Câu `print` chỉ là chuỗi tự do, muốn
>    lấy số phải viết regex và hỏng khi đổi câu chữ.
> 2. Giám sát, cảnh báo tự động: nhờ `event`, `level`, `timestamp` chuẩn hóa,
>    tôi đặt được alert khi chi phí tăng đột biến hoặc số dòng `ask_completed`
>    tụt về 0, và lần theo một `user_id` để tái hiện sự cố.

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
| 1 stage (bản đầu) | 1.125 MB |
| Multi-stage | 64 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> (Điền số đo thật từ `docker images` vào bảng trên.)
>
> Phần chênh lệch là những thứ chỉ cần lúc build mà không cần lúc chạy: cache
> và file tạm của pip, công cụ build (compiler, header), và các layer trung gian
> nằm lại trong image một stage. Multi-stage chỉ copy thư mục `/install` (thư
> viện đã cài) sang stage runtime nên mọi thứ còn lại bị bỏ đi.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sửa một ký tự trong `app/main.py`: stage builder (`COPY requirements.txt` và
> `pip install`) dùng lại từ cache vì `requirements.txt` không đổi. Ở stage
> runtime, base image, `ENV`, `WORKDIR` và `COPY --from=builder` vẫn từ cache.
> Layer `COPY . .` bị vô hiệu vì nội dung đã đổi, nên nó và các layer phía sau
> (`RUN useradd`, `USER`, `HEALTHCHECK`, `CMD`) phải chạy lại.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`: mọi lần sửa code làm layer
> `COPY . .` đổi, kéo theo `pip install` chạy lại từ đầu, tải và cài lại toàn
> bộ thư viện dù `requirements.txt` không đổi. Build từ vài giây thành hàng phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python có lỗ hổng, ví dụ injection cho phép chạy lệnh
> tùy ý; (2) kẻ tấn công chạy lệnh trong container với quyền của tiến trình app;
> (3) nếu app chạy bằng root thì đó là root trong container, có thể đọc mọi file,
> cài công cụ, sửa hệ thống; (4) từ đó lợi dụng lỗi container escape, mount
> nhạy cảm hoặc kernel bug để thoát ra host, và root trong container có thể
> ánh xạ thành quyền cao trên host.
>
> Lệnh `USER appuser` cắt chuỗi ở bước (3): tiến trình chạy bằng user thường
> không ghi được vào hệ thống, không cài được gói, và nhiều kỹ thuật thoát
> container cần quyền root nên không dùng được. Lỗ hổng vẫn còn nhưng thiệt hại
> bị giới hạn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây. Người dùng gửi 10 request ở giây 59 của phút
> hiện tại (dùng hết hạn mức của phút đó), rồi đến giây 00 của phút kế tiếp bộ
> đếm reset, họ gửi thêm 10 request nữa. Tổng 20 request trong khoảng 2 giây,
> gấp đôi hạn mức. Sliding window 60 giây không bị lỗi này vì luôn đếm 60 giây
> gần nhất tính từ thời điểm hiện tại, nên 10 request ở giây 59 vẫn còn tính.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất (bao nhiêu request trong một khoảng thời gian),
> bảo vệ hệ thống khỏi bị dồn dập. Cost guard giới hạn tổng chi phí tích lũy
> (USD) của mỗi người dùng, bảo vệ ví tiền.
>
> - Rate limit cho qua nhưng cost guard chặn: một người gửi đều đặn 1 request
>   mỗi phút, chưa bao giờ vượt hạn mức tần suất, nhưng mỗi câu hỏi rất dài và
>   tốn nhiều token nên cuối ngày tổng chi phí vượt ngân sách. Cost guard trả 402.
> - Rate limit chặn nhưng cost guard cho qua: một script gửi 100 câu hỏi rất
>   ngắn trong 1 giây. Tổng tiền chỉ vài cent, còn dưới ngân sách, nhưng vượt
>   hạn mức tần suất nên bị trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện khi gộp làm một endpoint có kiểm tra Redis:
> 1. Redis mất kết nối.
> 2. Endpoint gộp trả lỗi (503) trên cả 3 container cùng lúc, vì cả 3 cùng phụ
>    thuộc Redis.
> 3. Health check liên tiếp thất bại, vượt ngưỡng retries.
> 4. Orchestrator coi cả 3 container là "chết" và restart chúng (hoặc load
>    balancer loại cả 3 khỏi pool).
> 5. Cả cụm không còn instance nào phục vụ: downtime toàn phần, dù process app
>    vẫn khỏe và restart không sửa được Redis.
> 6. Khi Redis quay lại sau 30 giây, các container còn đang khởi động lại nên
>    downtime kéo dài hơn sự cố gốc.
>
> Tách ra thì `/health` (không gọi Redis) vẫn 200 nên không ai bị restart, còn
> `/ready` trả 503 để load balancer tạm ngừng gửi traffic, và khi Redis lại thì
> tự nhận traffic trở lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, `history_length` tăng đều theo từng lần gọi (0, 2, 4, 6, ...; mỗi
> lượt lưu một câu hỏi và một câu trả lời) dù request rơi vào container nào.
>
> Với dict Python trong bộ nhớ, mỗi container có dict riêng. Load balancer chia
> request vòng tròn cho 3 container nên con số nhảy lung tung, ví dụ 0, 0, 0,
> 2, 2, 2, 4, ... hoặc bị reset về 0 khi request rơi vào container chưa từng
> thấy user đó. Container restart cũng làm mất toàn bộ lịch sử. Đó là lý do
> app phải stateless: trạng thái nằm ở Redis dùng chung.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Tôi đã từng deploy lên gcp sai tên hay ký tự, rồi tên trong biến .env hay config settings.json của asp.net khiến việc deploy bắt đầu lại từ đâu build docker push docker lên cloud registry, deploy lên cloud run...
