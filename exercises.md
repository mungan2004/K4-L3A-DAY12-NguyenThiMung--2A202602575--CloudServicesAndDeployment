# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng đó bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thị Mừng  Mã học viên: 2A202602575

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định "changeme", khi deploy lên cloud mà quên set biến AGENT_API_KEY, app vẫn chạy. Kẻ tấn công có thể dễ dàng đoán ra key là "changeme" (hoặc dò tìm trên github) và dùng nó để gọi API thoải mái, dẫn tới thiệt hại lớn về tiền bạc API (do bị trừ tiền AI token). Việc "chết sớm" giúp ứng dụng cảnh báo ngay lập tức việc cấu hình thiếu sót từ lúc mới khởi động, giúp ta sửa lỗi kịp thời trước khi hệ thống mở cửa.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

`{"time": "2026-09-28T17:00:00Z", "level": "INFO", "event": "ask_completed", "user_id": "sv-test", "cost": 0.005}`

Hai việc log JSON làm được mà print thường không làm được:
1. Dễ dàng đưa vào các hệ thống quản lý Log (như Elasticsearch, Datadog) để lọc các log lỗi hoặc tính tổng chi phí (cost) theo từng user_id bằng câu lệnh truy vấn.
2. Dễ dàng dùng các công cụ (như `jq`) để phân tích tự động (machine-readable) thay vì phải viết regex lằng nhằng để tách chuỗi chữ.

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
| 1 stage (bản đầu) | ~ 900 MB |
| Multi-stage | ~ 150 MB |

Giải thích: Phần dung lượng chênh lệch khổng lồ đó là bộ công cụ build (gcc, c++ compiler), pip cache, và mã nguồn C của các thư viện trung gian. Khi dùng Multi-stage, ta chỉ copy "thành phẩm cuối cùng" (các file đã biên dịch) sang một base image gọn nhẹ (slim/alpine), vứt bỏ hoàn toàn các công cụ build nặng nề không cần thiết lúc chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Layer `COPY requirements.txt .` và `RUN pip install` vẫn được tái sử dụng (cache) do file requirements.txt không thay đổi.
- Layer `COPY . .` và các lệnh sau đó (như CMD) sẽ bị phá cache và phải chạy lại vì nội dung file main.py thay đổi.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Do toàn bộ source code bị copy vào sớm, nên việc sửa bất cứ file code nào cũng sẽ làm bước này thay đổi, kéo theo toàn bộ các bước sau nó (kể cả `RUN pip install`) đều bị mất cache và phải cài lại toàn bộ thư viện từ đầu. Quá trình build sẽ cực kỳ chậm.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Có lỗ hổng trong code (ví dụ lỗi RCE) -> Kẻ tấn công thực thi được script độc hại bên trong container với quyền root -> Do container chia sẻ chung Kernel với máy host, kẻ tấn công khai thác tiếp lỗ hổng Kernel (Container Escape) -> Thoát ra ngoài và chiếm toàn quyền hệ thống thật (Host) bằng quyền root.

Lệnh `USER` cắt đứt chuỗi ở ngay bước đầu: Kẻ tấn công chui vào được container nhưng chỉ có quyền của user thường (hạn chế). Họ không thể chạy các lệnh đặc quyền, không cài được mã độc sâu, khiến kỹ thuật leo thang đặc quyền để thoát ra máy host khó khăn hơn rất nhiều.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Họ có thể gửi tối đa **20 request** trong 2 giây liên tiếp.

Giải thích: Cách đếm fixed window (reset lúc phút:00) có kẽ hở lớn ở điểm giao thoa. Nếu họ gửi 10 request ở giây 59 của phút thứ 1 (ngay trước khi reset), hệ thống cho qua vì chưa chạm hạn mức phút 1. Tới giây 00 của phút thứ 2, bộ đếm bị reset về 0, họ lập tức gửi thêm 10 request nữa và vẫn được cho qua. Kết quả là trong 2 giây liên tiếp (giây 59 và 00), API phải hứng chịu gấp đôi hạn mức (20 req) so với thiết kế, làm tăng nguy cơ quá tải máy chủ. Cửa sổ trượt khắc phục triệt để bằng cách luôn xét liên tục 60 giây lùi từ mốc hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Rate limit:** Giới hạn TẦN SUẤT (tốc độ gọi API, VD 10 req/phút) để chống DDoS, spam, giúp server không bị sập. Dùng bộ đếm xoá đi làm lại liên tục.
- **Cost guard:** Giới hạn TỔNG TIỀN (tích luỹ, VD 10$/tháng) để chống cháy túi người phát triển. Dùng bộ đếm cộng dồn suốt 1 chu kỳ dài.

Tình huống Rate limit cho qua, Cost guard chặn: Một người dùng cực kỳ ngoan ngoãn, mỗi ngày chỉ hỏi AI đúng 1 câu (chắc chắn dưới mức 10/phút của rate limit). Nhưng họ hỏi những câu cực kỳ dài và liên tục trong 30 ngày khiến tổng tiền tiêu tốn vượt quá 10$. Hệ thống cost guard sẽ nhảy ra chặn.

Tình huống ngược lại (Rate limit chặn, Cost guard cho): Một người dùng mới toanh chưa tiêu đồng nào (cost = 0), nhưng vì nôn nóng nên bấm F5 liên tục gửi 20 câu hỏi trong 1 giây. Lúc này rate limit sẽ lập tức chặn lại để bảo vệ server, dù người dùng vẫn còn đầy tiền.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện (crashloop thảm họa):
1. Giây 0: Redis bị đứt kết nối mạng.
2. Giây 5: K8s/Docker tự động gọi vào /health để kiểm tra sức khoẻ. Vì gộp chung code, /health cố gọi Redis và thất bại, trả về HTTP 503.
3. Giây 10: Docker thấy /health báo 503 bèn kết luận "Container này đã chết/treo", lập tức Kill (tiêu diệt) và ép khởi động lại toàn bộ 3 container.
4. Giây 20: Container vừa khởi động lên lại bị kiểm tra /health, Redis vẫn chưa có lại -> lại báo 503 -> Lại bị Kill. Vòng lặp restart diễn ra liên tục.

Hậu quả: Ứng dụng sập hoàn toàn. Việc tách biệt /ready giúp ứng dụng chỉ báo "Chưa sẵn sàng nhận khách" (ngắt luồng traffic) nhưng /health vẫn 200 (app vẫn sống) nên K8s không điên cuồng restart lại nó. Tới khi Redis sống lại, /ready xanh lại tự động hứng traffic bình thường.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu dùng biến dict Python (bộ nhớ trong của từng container), do có 3 container chạy song song, mỗi container sẽ tự giữ một cuốn sổ dict riêng biệt. Khi load balancer phân phối các request ngẫu nhiên tới 3 container này, lịch sử của user sẽ bị phân mảnh. 
Hậu quả là `history_length` sẽ nhảy số lộn xộn (ví dụ: 1 -> 2 -> 1 -> 3 -> 2) thay vì tăng dần đều, do câu hỏi rơi trúng container chưa từng trò chuyện với user đó. Redis giải quyết việc này bằng cách tạo ra một "bộ não dùng chung" cho cả 3 container.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi thường gặp: App trả về 503 lúc khởi động vì không kết nối được Redis do cấu hình thiếu/sai REDIS_URL.
- Thông báo lỗi (hoặc biểu hiện): /ready trả về {"status":"not ready","redis":false}, check log ghi "Redis connection failed".
- Tìm nguyên nhân: Đọc log của Railway, phát hiện ra hàm ping Redis trả False. Do app đang chạy mặc định với chuỗi `redis://localhost:6379`, nhưng trên cloud Redis nằm ở địa chỉ khác.
- Cách sửa: Vào tab Variables của project trên Railway, tạo biến mới tên `REDIS_URL`, và trỏ nó vào Database Redis (chọn Reference variable hoặc gõ giá trị `${{Redis.REDIS_URL}}`). Railway sẽ tự động nạp lại app với đúng URL.
