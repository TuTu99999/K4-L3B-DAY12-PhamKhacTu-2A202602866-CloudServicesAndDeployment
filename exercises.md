# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Họ và tên: **Phạm Khắc Tú**  Mã học viên: **2A202602866**

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy Render mà quên khai báo `AGENT_API_KEY`, ứng dụng sẽ báo lỗi validation ngay lúc khởi động và health check không thể báo thành công. Nhờ vậy tôi biết cấu hình secret đang thiếu trước khi public URL nhận traffic. Nếu dùng mặc định `changeme`, service vẫn chạy và người khác có thể đoán đúng khóa này để gọi `/ask`, làm tốn quota và chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")` không làm được.

> Dòng log tôi nhận được là `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T03:03:11.847962+00:00","user_id":"cp4-runtime-check","tokens_in":4,"tokens_out":38,"cost_usd":2.34e-05}`. Từ các field rõ ràng này, tôi có thể (1) lọc hoặc nhóm log theo `event`, `user_id` và khoảng thời gian; (2) cộng `tokens_in`, `tokens_out`, `cost_usd` để làm biểu đồ hoặc cảnh báo chi phí. Chuỗi `print` thường không có cấu trúc để hệ thống log xử lý tự động như vậy.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

| Bản | Dung lượng |
|-----|-----------|
| 1 stage dùng `python:3.11` | **1.73 GB** |
| Multi-stage dùng `python:3.11-slim` | **271 MB** |

> Tôi build lại đúng Dockerfile một-stage ban đầu và đo bằng `docker images`: bản một-stage là **1.73 GB**, còn bản multi-stage production là **271 MB**. Bản production giảm khoảng **1.46 GB**, tương đương khoảng **84%**. Phần chênh lệch chủ yếu đến từ base image `python:3.11` đầy đủ chứa nhiều gói hệ điều hành và công cụ không cần cho lúc chạy. Image multi-stage dùng base `slim`; stage `builder` chỉ tạo các dependency rồi runtime chỉ nhận phần đã cài cùng source code. Vì vậy các công cụ build và layer trung gian không đi vào image cuối. Image hiện tại vẫn có thể giảm thêm vì `requirements.txt` còn chứa dependency phục vụ test.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt `COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `WORKDIR`, `COPY requirements.txt`, cài dependency và tạo user đều được lấy từ cache. Layer copy source `app/` và các layer sau nó phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, một thay đổi nhỏ trong source cũng làm layer copy đổi, kéo theo bước cài toàn bộ dependency chạy lại dù `requirements.txt` không đổi; build sẽ chậm hơn và cache kém hiệu quả.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng Python có thể cho kẻ tấn công thực thi lệnh trong process Uvicorn. Nếu process là root, họ có quyền root trong container, có thể sửa mọi file, dùng thiết bị hoặc mount nhạy cảm được cấp cho container rồi tìm cách khai thác kernel hay Docker socket để tác động lên host. `USER appuser` làm process chạy bằng UID 10001 không đặc quyền, nên ngay sau bước chiếm process, quyền ghi file và capability đã bị giới hạn. Cách này giảm mạnh phạm vi thiệt hại, dù vẫn phải tránh mount Docker socket và cập nhật dependency.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được con số đó.

> Có thể gửi **20 request**: gửi 10 request trong giây cuối của phút cũ, rồi khi bộ đếm reset ở giây 00, gửi tiếp 10 request trong giây đầu của phút mới. Sliding window nhìn lại đúng 60 giây nên cả hai nhóm vẫn nằm trong cùng cửa sổ và không cho phép burst 20 request như cách đếm theo phút đồng hồ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit kiểm soát tốc độ gọi trong 60 giây, còn cost guard kiểm soát tổng tiền theo user trong tháng UTC. Một user chỉ gửi một request sau thời gian nghỉ nên rate limit cho qua, nhưng chi phí tháng đã gần hết và chi phí ước tính làm vượt ngân sách thì cost guard trả 402. Ngược lại, một user còn rất nhiều ngân sách nhưng gửi hơn 10 câu hỏi rất ngắn trong 60 giây thì cost guard vẫn cho phép về chi phí, còn rate limiter trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm 3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối → endpoint chung của cả ba container trả 503 → orchestrator coi cả ba process bị lỗi → cả ba bị restart dù bản thân Uvicorn vẫn sống → cụm tạm thời không còn instance ổn định và có thể lặp restart trong suốt 30 giây Redis lỗi. Khi tách endpoint, `/health` vẫn trả 200 nên container không bị restart, còn `/ready` trả 503 để load balancer tạm ngừng gửi request. Redis phục hồi thì `/ready` tự trở lại 200 và cụm nhận traffic tiếp.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một `X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Lệnh scale trực tiếp của tôi gặp lỗi `port is already allocated` vì service đang map cố định `8000:8000`; muốn chạy ba replica cần đặt load balancer phía trước và không publish cùng một host port cho từng replica. Phần chia sẻ state được kiểm tra với cùng Redis: hai lần gọi liên tiếp cho cùng user cho `history_length` lần lượt là 0 rồi 2, và test hai store instance cũng đọc cùng lịch sử. Khi scale đúng qua load balancer, Redis sẽ giữ dãy 0, 2, 4… dù request rơi vào container nào. Nếu dùng dict Python, mỗi container có lịch sử riêng nên số có thể lặp hoặc lùi như 0, 0, 2, 0 tùy container nhận request; restart container còn làm lịch sử của instance đó mất hẳn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud: thông báo lỗi là gì, bạn tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi Render báo service `Live`, lần đầu tôi kiểm tra `/ask` lại nhận `422 Unprocessable Entity` với chi tiết `json_invalid`, thay vì 401 như dự kiến. `/health` và `/ready` đều 200 nên tôi biết deploy và Redis không hỏng; đọc body lỗi cho thấy JSON từ lệnh `curl.exe` trong PowerShell bị sai do cách đặt dấu nháy. Tôi sửa bằng cách tạo body bằng `ConvertTo-Json` rồi gửi lại request. Server đọc được JSON và trả đúng 401 khi thiếu `X-API-Key`.
