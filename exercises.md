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

> *Câu trả lời của bạn*

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

> *Câu trả lời của bạn*

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> *Câu trả lời của bạn*

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
