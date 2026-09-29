# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Doãn Hữu Nguyên  Mã học viên: 2A202602671

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Giả sử em deploy lên Railway mà quên set biến `AGENT_API_KEY` trong tab Variables.
> Vì `agent_api_key` không có mặc định nên `Settings()` ném `ValidationError` ngay lúc khởi động, container không lên được, health check fail và Railway báo deploy lỗi, bản cũ vẫn chạy tiếp.
> Em thấy lỗi ngay trong lúc đang ngồi xem log deploy nên vào set biến rồi deploy lại là xong.
> Nếu để mặc định `"changeme"` thì app vẫn chạy bình thường, `/ask` mở ra public với cái khóa nằm sẵn trong source trên repo public, ai đọc code cũng gọi được.
> Trường hợp đó em sẽ không thấy lỗi gì cả, chỉ biết khi nhìn hóa đơn LLM tăng bất thường.
> Em có thử chạy `python -c "from app.config import Settings; Settings(_env_file=None)"` (dùng `_env_file=None` để bỏ qua file `.env`, giống môi trường cloud chưa set biến) và nhận được `ValidationError: 1 validation error for Settings` với dòng `agent_api_key  Field required [type=missing, input_value={}, input_type=dict]`.
> Nghĩa là thiếu secret thì app từ chối chạy luôn, lỗi lộ ra lúc deploy chứ không phải lúc đã bị người khác dùng chùa.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Em chạy stack bằng `docker compose up -d --build`, gọi `/ask` với `X-User-Id: sv01` rồi xem `docker compose logs agent`, thu được dòng này:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:05:33.474646+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}`
> Việc thứ nhất: mỗi dòng là một JSON có sẵn `user_id` và `cost_usd`, nên em lọc được theo user rồi cộng `cost_usd` để biết ai tiêu nhiều tiền nhất trong ngày.
> Ngay trong lần test, em lọc theo `"event": "ask_completed"` và đếm được 11 dòng (1 của sv01, 10 của sv-rl), khớp đúng số request trả 200.
> Việc thứ hai: có `level` và `timestamp` theo chuẩn ISO UTC, nên hệ thống log trên cloud đếm được số dòng `"level": "error"` trong 5 phút gần nhất để bắn cảnh báo, kể cả khi log gom từ nhiều container.
> Còn `print("đã trả lời xong")` chỉ là một câu chữ, không biết của user nào, tốn bao nhiêu, lúc nào, muốn thống kê thì phải tự viết regex đoán.

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
| 1 stage (bản đầu) | 1.73 GB (1730 MB) |
| Multi-stage | 271 MB |

*Số lấy từ cột DISK USAGE của `docker images`.*

Giải thích: phần dung lượng chênh lệch đó là những gì?

> 1 stage 1.73 GB, multi-stage 271 MB, nhỏ hơn khoảng 6 lần.
> Phần lớn chênh lệch nằm ở base image: `python:3.11` bản đầy đủ mang theo gcc, header C, rất nhiều thư viện hệ thống và công cụ build, còn `python:3.11-slim` chỉ giữ những gì cần để chạy Python.
> Ở bản multi-stage, stage builder cài thư viện vào `/install` rồi stage runtime chỉ copy đúng thư mục đó sang, nên những thứ chỉ dùng lúc build bị bỏ lại hết.
> Bản single còn chạy `pip install` không có `--no-cache-dir` nên cache của pip nằm luôn trong image, và `COPY . .` kéo theo cả tài liệu, `grade.py`... dù `.dockerignore` đã chặn `.env`, `.venv`, `.git`, `tests`.
> Image multi vẫn chưa gọn nhất vì `requirements.txt` đang gộp cả thư viện test (pytest, fakeredis, httpx, PyYAML), nếu tách ra `requirements-dev.txt` thì image còn nhỏ hơn nữa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Em thêm một dòng comment vào cuối `app/main.py` rồi build lại cả hai bản để so sánh.
> Với bản multi, các layer `WORKDIR /build`, `COPY requirements.txt`, `RUN pip install`, `RUN useradd`, `WORKDIR /app`, `COPY --from=builder /install /usr/local` đều báo `CACHED`, chỉ `COPY app ./app` và `COPY utils ./utils` chạy lại, cả lần build mất 1.15 s.
> Lý do là Docker cache theo từng layer: layer đầu tiên có input thay đổi là `COPY app`, nên nó và mọi layer phía sau đều chạy lại, kể cả `COPY utils` dù `utils` không đổi gì.
> Bản single (`Dockerfile.single`) đặt `COPY . .` trước `RUN pip install`, nên sửa một dòng comment cũng làm layer COPY thay đổi và kéo theo `pip install` chạy lại toàn bộ: bước này mất 152.1 s, cả lần build mất 157.4 s.
> Lần đo này pip còn gặp `ReadTimeoutError` khi tải từ `files.pythonhosted.org` và phải retry, tức là mỗi lần sửa code ở bản single đều phụ thuộc vào mạng.
> Tính ra bản single chậm hơn khoảng 137 lần (157.4 s so với 1.15 s), và kể cả khi mạng tốt thì mỗi lần sửa code nó vẫn phải tải và cài lại toàn bộ thư viện, còn bản multi thì không.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Giả sử code Python có lỗ hổng kiểu command injection hoặc RCE, kẻ tấn công sẽ chạy được lệnh bên trong container với đúng quyền của process app.
> Nếu container chạy root thì process đó là uid 0, mà container dùng chung kernel với host nên uid 0 trong container cũng chính là uid 0 trên host (khi không bật user namespace).
> Lúc này chỉ cần thêm một lỗ hổng container escape, hoặc container lỡ được mount thư mục của host hay `docker.sock`, là kẻ tấn công thành root trên máy host.
> Lệnh `USER appuser` cắt chuỗi này ngay ở bước thứ hai, vì lệnh của kẻ tấn công chỉ chạy với uid 10001: em chạy `docker compose exec agent id` thì ra `uid=10001(appuser) gid=10001(appuser) groups=10001(appuser)`.
> Với user này, kẻ tấn công không sửa được file hệ thống, cũng không sửa được code app vì code thuộc root và chỉ đọc được, nên muốn leo thang quyền khó hơn nhiều.
> Kể cả khi thoát được ra ngoài container thì cũng chỉ là một user thường trên host, không phải root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Nếu đếm theo phút đồng hồ thì trong 2 giây liên tiếp một user gửi được tối đa 20 request.
> Cách làm là gửi 10 request lúc hh:mm:59, cả 10 đều hợp lệ; sang hh:(mm+1):00 bộ đếm reset về 0 nên gửi tiếp 10 request nữa vẫn hợp lệ.
> Tức là hạn mức "10/phút" thực chất cho gấp đôi ngay tại ranh giới giữa hai phút.
> Sliding window của em không có kẽ hở này vì mỗi lần check nó đếm số request trong 60 giây tính ngược từ hiện tại (`zremrangebyscore` xóa những request cũ hơn `now - 60` rồi `zcard`), nên 10 request lúc :59 vẫn còn nằm trong cửa sổ ở giây :00.
> Em test tay 15 request liên tiếp với `X-User-Id: sv-rl` và nhận được dãy `200 200 200 200 200 200 200 200 200 200 429 429 429 429 429`, các request bị chặn có header `Retry-After: 60`.
> Đúng 10 request đầu được qua vì em đếm trước rồi mới ghi, và request bị 429 không được ghi vào ZSET nên spam thêm cũng không kéo dài thời gian bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lượng request trong một khoảng ngắn (10 request/60 giây), trả 429 và đợi một lúc là gọi lại được.
> Cost guard giới hạn số tiền cộng dồn trong cả tháng theo từng user, trả 402 và chỉ hết chặn khi sang tháng mới hoặc được nâng ngân sách.
> Tình huống rate limit cho qua nhưng cost guard chặn: em chạy agent với `MONTHLY_BUDGET_USD=0.0001` (đặt bằng biến shell, không sửa `.env`) rồi gọi `/ask` 10 lần liên tiếp với `X-User-Id: sv-budget`, nhận được `200 200 200 200 402 402 402 402 402 402`.
> Chi phí 4 lần đầu tăng dần 1.995e-05, 3.105e-05, 3.765e-05, 4.44e-05 USD vì lịch sử hội thoại được gửi kèm, tổng 0.00013305 USD đã vượt ngân sách nên từ request thứ 5 bị 402, trong khi cả 10 request vẫn nằm trong hạn mức 10/phút nên không cái nào bị 429.
> Em để ý request thứ 4 vẫn được cho qua vì lúc check tổng mới là 8.865e-05 ≤ 0.0001 (check dùng `estimated_cost=0`), cộng xong thì tổng thành 1.3305e-04, vượt ngân sách khoảng 33%; muốn chặt hơn thì phải ước lượng chi phí trước rồi truyền vào `estimated_cost`.
> Với LLM thật cũng vậy: một user gọi thong thả 5 request/phút nhưng mỗi request hàng chục nghìn token thì chỉ vài ngày là cháy ngân sách.
> Tình huống ngược lại: một script lỗi gọi `/ask` 15 lần liên tiếp với câu "test", mỗi lần chỉ tốn khoảng 2e-05 USD nên ngân sách 10 USD gần như không đổi, nhưng rate limit vẫn chặn từ request thứ 11 như em thấy ở Câu 6.
> Vì vậy cần cả hai: rate limit chống spam và bảo vệ server, cost guard chống cháy túi.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Câu trả lời của bạn*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Câu trả lời của bạn*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*
