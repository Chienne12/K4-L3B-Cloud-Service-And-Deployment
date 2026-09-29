# Checklist hoàn thành bài Cloud Service & Deployment

Luồng thực hiện:

`Setup → CP1 Config → CP2 Docker → CP3 Security → CP4 Reliability → CP5 Deploy → Hoàn thiện hồ sơ → Nộp bài`

> Làm lần lượt từng checkpoint. Sau mỗi checkpoint: chạy test tương ứng, sửa hết lỗi, sau đó commit trước khi chuyển sang phần tiếp theo.

---

## 0. Setup môi trường

- [x] Đổi tên repository đúng mẫu:
  `K4-L3B-DAY12-<HoVaTen>-<MSSV>-CloudServicesAndDeployment`.
- [x] Tạo và kích hoạt môi trường ảo `.venv`.
- [x] Chạy `pip install -r requirements.txt`.
- [x] Copy `.env.example` thành `.env`.
- [x] Sinh `AGENT_API_KEY` riêng, không dùng giá trị mẫu.
- [x] Khởi động Redis bằng `docker compose up -d redis` hoặc tạm dùng `REDIS_URL=fake://`.
- [x] Xác nhận `.env` không được Git theo dõi.
- [x] Chạy thử:

```powershell
python -m pytest tests/ -v -m "not docker"
```

Kết quả cần đạt: pytest chạy được, không có `ModuleNotFoundError`. Test chưa pass ở bước này là bình thường.

---

## 1. CP1 — Config, Logging và Health

### `app/config.py`

Mục đích: đọc toàn bộ cấu hình từ biến môi trường và dừng sớm nếu thiếu secret.

- [x] Khai báo `port: int = 8000`.
- [x] Khai báo `agent_api_key: str` và không đặt giá trị mặc định.
- [x] Khai báo `redis_url: str = "redis://localhost:6379/0"`.
- [x] Khai báo `rate_limit_per_minute: int = 10`.
- [x] Khai báo `monthly_budget_usd: float = 10.0`.
- [x] Khai báo `log_level: str = "INFO"`.
- [x] Không hardcode API key hoặc secret trong code.

Luồng: biến môi trường → `Settings` kiểm tra kiểu → đối tượng cấu hình dùng chung.

### `app/logging_utils.py`

Mục đích: tạo log JSON một dòng để hệ thống cloud có thể tìm kiếm và thống kê.

- [x] Cài đặt `log_event()`.
- [x] Tạo các trường `event`, `level`, `timestamp`.
- [x] Chuyển `level` thành chữ thường.
- [x] Gộp các trường bổ sung từ `**fields`.
- [x] Dùng `json.dumps(..., ensure_ascii=False)` và không dùng `indent`.
- [x] In đúng một dòng ra stdout.
- [x] Trả về chuỗi JSON vừa in.

Luồng: tên sự kiện + dữ liệu bổ sung → thêm timestamp → JSON một dòng → stdout.

### `app/main.py` — `/health`

Mục đích: cho platform biết process còn sống hay đang tắt.

- [x] Bình thường trả `status: ok`, tên service và version.
- [x] Khi `lifecycle.shutting_down` là `True`, trả HTTP 503 với `status: shutting_down`.
- [x] Không gọi Redis, database hoặc dùng dependency ngoài trong `/health`.

Luồng: đọc trạng thái process → trả 200 hoặc 503 → kết thúc, không kiểm tra Redis.

### Kiểm tra CP1

```powershell
python -m pytest tests/test_cp1.py -v
```

- [x] Toàn bộ test CP1 pass.
- [x] Commit checkpoint CP1.

---

## 2. CP2 — Docker

### `Dockerfile`

Mục đích: tạo image nhỏ, an toàn và chạy đúng trên cloud.

- [x] Dùng multi-stage build và có stage `builder`.
- [x] Dùng base image gọn như `python:3.11-slim`.
- [x] Copy `requirements.txt` và cài dependency trước khi copy source.
- [x] Runtime stage chỉ nhận dependency và source cần thiết.
- [x] Tạo user thường và chuyển sang bằng lệnh `USER`.
- [x] Thêm `HEALTHCHECK` gọi `/health`.
- [x] Lệnh chạy Uvicorn đọc cổng từ biến `$PORT`.
- [x] Không hardcode API key hoặc secret.

Luồng: builder cài dependency → runtime nhận kết quả → user thường chạy Uvicorn bằng `$PORT`.

### `docker-compose.yml`

Mục đích: chạy đầy đủ stack `agent + Redis` trên máy local.

- [x] Thêm service `agent`.
- [x] Build agent từ Dockerfile trong thư mục hiện tại.
- [x] Map cổng `8000:8000`.
- [x] Truyền API key bằng `${AGENT_API_KEY}`.
- [x] Đặt `REDIS_URL=redis://redis:6379/0`.
- [x] Khai báo agent phụ thuộc service `redis`.
- [x] Thêm healthcheck gọi `/health`.
- [x] Không ghi trực tiếp giá trị secret trong Compose.

Luồng: Compose tạo Redis → hostname là `redis` → agent kết nối qua mạng nội bộ.

### `.dockerignore`

Mục đích: loại file không cần thiết và ngăn secret lọt vào Docker image.

- [x] Loại `.env`.
- [x] Loại `.git` và `.gitignore`.
- [x] Loại `.venv`, `venv`.
- [ ] Loại `__pycache__`, `.pytest_cache`, `*.pyc`.
- [x] Không loại nhầm `app/`, `utils/` hoặc `requirements.txt`.

### Kiểm tra CP2

```powershell
python -m pytest tests/test_cp2.py -v
docker build -t day12-agent:prod .
docker compose up -d
curl.exe http://localhost:8000/health
```

- [x] Toàn bộ test CP2 pass.
- [x] Container agent và Redis đều chạy hoặc healthy.
- [x] Ghi lại kích thước image để trả lời `exercises.md`.
- [x] Commit checkpoint CP2.

---

## 3. CP3 — API Security

### `app/auth.py`

Mục đích: chỉ cho request có API key hợp lệ đi vào `/ask`.

- [x] Đọc key chuẩn từ `get_settings().agent_api_key`.
- [x] Thiếu hoặc sai `X-API-Key` thì trả HTTP 401.
- [x] Dùng `secrets.compare_digest()`, không dùng `==`.
- [x] Key hợp lệ thì trả `X-User-Id`.
- [x] Không có `X-User-Id` thì trả `anonymous`.

Luồng: đọc header → so sánh key → sai thì dừng 401 → đúng thì trả `user_id`.

### `app/rate_limiter.py`

Mục đích: giới hạn số request của từng user trong 60 giây gần nhất.

- [x] `hit_count()` xóa các entry cũ hơn 60 giây.
- [x] `hit_count()` trả số request còn lại trong sorted set.
- [x] `check()` kiểm tra số lượt trước khi ghi request mới.
- [x] Quá giới hạn thì trả HTTP 429 và header `Retry-After`.
- [x] Dùng member duy nhất, kết hợp timestamp với UUID.
- [x] Đặt TTL 60 giây cho key.

Luồng: khoanh vùng 60 giây → xóa mốc cũ → đếm → quá giới hạn thì dừng → chưa quá thì ghi mốc mới.

### `app/cost_guard.py`

Mục đích: giới hạn tổng chi phí của từng user theo tháng.

- [x] `spent()` trả `0.0` khi Redis chưa có dữ liệu.
- [x] `spent()` ép dữ liệu Redis thành `float`.
- [x] `check()` trả HTTP 402 nếu tổng dự kiến vượt ngân sách.
- [x] `record()` cộng chi phí bằng `incrbyfloat`.
- [x] `record()` đặt TTL và trả tổng chi phí mới.
- [x] Dữ liệu được tách theo user và tháng.

Luồng: user + tháng → đọc tổng tiền → kiểm tra budget → gọi LLM → ghi chi phí thật.

### `app/main.py` — `/ask`

Mục đích: ráp các lớp bảo vệ, lịch sử và mock LLM thành endpoint chính.

- [x] Thực hiện đúng thứ tự:
  1. `limiter.check(user_id)`.
  2. `guard.check(user_id)`.
  3. Đọc history.
  4. Gọi mock LLM.
  5. Lưu câu hỏi và câu trả lời.
  6. Ghi nhận chi phí.
  7. Ghi log hoàn thành.
  8. Trả response.
- [x] Response có `answer`, `user_id`, `history_length`, `cost_usd`, `tokens`.
- [x] Không gọi LLM trước khi kiểm tra rate limit và ngân sách.

Luồng: auth → rate limit → cost guard → history → LLM → lưu state → ghi cost/log → response.

### Kiểm tra CP3

```powershell
python -m pytest tests/test_cp3.py -v
```

- [x] Toàn bộ test CP3 pass.
- [x] Commit checkpoint CP3.

---

## 4. CP4 — Stateless và Reliability

### `app/store.py`

Mục đích: lưu history trong Redis để nhiều instance dùng chung state.

- [ ] `ping()` trả `True` khi Redis hoạt động.
- [ ] `ping()` bắt exception và trả `False` khi Redis lỗi.
- [ ] `append()` lưu message JSON bằng `rpush`.
- [ ] Dùng `ltrim` để chỉ giữ 20 message gần nhất.
- [ ] Đặt TTL history là 7 ngày.
- [ ] `get_history()` đọc Redis list và `json.loads` từng message.
- [ ] Chưa có history thì trả list rỗng.
- [ ] Không tạo dict hoặc list toàn cục để giữ state.

Luồng: `user_id` → Redis key → nối message → cắt phần cũ → đặt TTL → mọi instance đọc cùng history.

### `app/lifecycle.py`

Mục đích: nhận tín hiệu shutdown và dừng service mà không cắt ngang request.

- [ ] `request_shutdown()` đặt `shutting_down=True`.
- [ ] Gọi lại handler cũ nếu handler đó callable.
- [ ] `install()` lưu handler cũ trước khi ghi đè.
- [ ] Đăng ký handler cho cả `SIGTERM` và `SIGINT`.

Luồng: nhận signal → bật cờ shutdown → health/readiness trả 503 → chuyển tiếp handler cũ → Uvicorn thoát.

### `app/main.py` — `/ready`

Mục đích: cho load balancer biết instance có đủ điều kiện nhận traffic hay không.

- [ ] Đang shutdown thì trả HTTP 503.
- [ ] Redis không ping được thì trả HTTP 503 với `redis: false`.
- [ ] Redis hoạt động thì trả HTTP 200 với `status: ready`.
- [ ] Xác nhận `/health` vẫn không phụ thuộc Redis.

Luồng: kiểm tra shutdown → ping Redis → lỗi thì 503 → thành công thì ready 200.

### Kiểm tra CP4

```powershell
python -m pytest tests/test_cp4.py -v
```

- [ ] Toàn bộ test CP4 pass.
- [ ] Commit checkpoint CP4.

---

## 5. CP5 — Cloud Deployment

### `railway.toml` hoặc `render.yaml`

Mục đích: cấu hình cách platform build, chạy và kiểm tra service.

- [ ] Chọn Railway hoặc Render.
- [ ] Chỉ chỉnh file của platform đã chọn nếu cần.
- [ ] Xác nhận build bằng Dockerfile.
- [ ] Xác nhận healthcheck gọi `/health`.
- [ ] Cấu hình `AGENT_API_KEY` trong dashboard/secret store.
- [ ] Cấu hình Redis thật và `REDIS_URL`.
- [ ] Cấu hình rate limit, budget và log level.
- [ ] Không commit giá trị secret.

Luồng: GitHub repo → platform build Docker image → inject environment → chạy app → healthcheck → cấp HTTPS URL.

### `DEPLOYMENT.md`

Mục đích: lưu thông tin và bằng chứng của bản deploy thật.

- [ ] Điền họ tên và mã học viên.
- [ ] Điền link repository đúng tên.
- [ ] Điền Public URL HTTPS thật.
- [ ] Ghi platform và ngày deploy.
- [ ] Liệt kê tên biến môi trường, không ghi giá trị secret.
- [ ] Chạy các lệnh kiểm tra `/health`, `/ready`, `/ask`.
- [ ] Dán output thật vào tài liệu.
- [ ] Xóa tất cả placeholder `(điền...)` và URL `TODO`.

### `screenshots/`

- [ ] Thêm `dashboard.png` chụp dashboard của platform.
- [ ] Thêm `health.png` chụp kết quả gọi `/health`.
- [ ] Kiểm tra ảnh không làm lộ API key hoặc secret.

### Kiểm tra CP5

```powershell
python -m pytest tests/test_cp5.py -v
```

- [ ] Toàn bộ test CP5 pass, hoặc ghi rõ phương án local fallback.
- [ ] Commit checkpoint CP5.

---

## 6. Hoàn thiện hồ sơ bài làm

### `exercises.md`

Mục đích: chứng minh đã chạy, quan sát và hiểu từng phần của bài.

- [ ] Điền họ tên và mã học viên.
- [ ] Trả lời đủ 10 câu bằng lời của mình.
- [ ] Câu 2 có log JSON lấy từ lần chạy thật.
- [ ] Câu 3 có kích thước image đo thật.
- [ ] Câu 4 mô tả layer cache sau khi thử build lại.
- [ ] Câu 9 dựa trên quan sát khi scale nhiều instance.
- [ ] Câu 10 ghi một lỗi deploy thực tế và cách sửa.
- [ ] Không còn dòng `*Câu trả lời của bạn*`.

### Các file không cần sửa

- `utils/mock_llm.py`: mock LLM đã cho sẵn.
- `tests/`: dùng để kiểm tra, không sửa test để làm test pass.
- `grade.py`: dùng để tính điểm, không cần sửa.
- `nginx/nginx.conf`: phần mở rộng, không bắt buộc.

---

## 7. Bonus CI/CD — chỉ làm sau CP1–CP5

### `.github/workflows/ci.yml`

Mục đích: tự động test, build và chỉ deploy code đã vượt qua kiểm tra.

- [ ] Workflow chạy khi `push` và `pull_request`.
- [ ] Có bước cài dependency từ `requirements.txt`.
- [ ] Có job chạy pytest nhưng không gọi test CP5 cần service public.
- [ ] Có bước build Docker image.
- [ ] Có job deploy với `needs` phụ thuộc job test/build.
- [ ] Chỉ deploy từ nhánh `main`.
- [ ] Token deploy lấy từ GitHub Secrets.
- [ ] Các GitHub Action được ghim version.
- [ ] Thêm badge CI vào `README.md`.

```powershell
python -m pytest tests/test_bonus_cicd.py -v
```

---

## 8. Kiểm tra cuối trước khi nộp

- [ ] Không còn `NotImplementedError` trong `app/`.
- [ ] Chạy toàn bộ test và biết rõ nguyên nhân của mọi test còn fail.
- [ ] Chạy `python grade.py` và kiểm tra điểm.
- [ ] Repo đúng tên quy định.
- [ ] `.env` không được Git theo dõi.
- [ ] Không có API key, token, mật khẩu hoặc private key trong repo.
- [ ] `DEPLOYMENT.md` không còn placeholder.
- [ ] `screenshots/` có đủ ảnh minh chứng.
- [ ] `exercises.md` đã trả lời đủ 10 câu.
- [ ] Có nhiều commit thể hiện tiến trình từng checkpoint.
- [ ] Repository GitHub ở chế độ public.
- [ ] Nộp đúng link repository lên Codelab.

Lệnh kiểm tra cuối:

```powershell
python -m pytest tests/ -v
python grade.py
git status --short
git ls-files
```

