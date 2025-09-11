# Bài thực hành Jenkins – CI/CD cho House Price API

Tài liệu này giúp bạn ôn tập, chạy và thực hành Jenkins cho dự án FastAPI dự đoán giá nhà. Bao gồm: chuẩn bị môi trường, khởi chạy Jenkins bằng Docker, cấu hình pipeline, đẩy image lên Docker Hub, triển khai và xử lý sự cố thường gặp.

---

## 1) Kiến trúc và luồng tổng quát

- GitHub: lưu code và `Jenkinsfile` (định nghĩa pipeline).
- Jenkins: clone code, chạy Test → Build → Push Docker image.
- Docker Hub: lưu image kết quả để các môi trường khác pull/deploy.

Luồng: Commit lên GitHub → Jenkins chạy pipeline → build image → push lên Docker Hub → server/developer pull image để chạy.

---

## 2) Cấu trúc repo liên quan

- `Jenkinsfile`: pipeline ở thư mục gốc (dùng cho Multibranch).
- `lesson11_CICD_jenkins/`:
  - `Dockerfile`: build ứng dụng FastAPI.
  - `requirements.txt`: dependencies (FastAPI, Uvicorn, pytest…).
  - `main.py`: API `POST /predict` dùng model `models/model.pkl`.
  - `tests/`: test đơn giản xác nhận model hoạt động.
  - `docker-compose.yaml`: chạy Jenkins (custom image) trong Docker.
  - `custom_jenkins/Dockerfile` và `custom_jenkins/plugins.txt`: image Jenkins kèm Docker CLI và plugin cần thiết (Docker Pipeline, Declarative, v.v.).

---

## 3) Yêu cầu môi trường

- Docker Desktop/Engine và Docker Compose.
- Tài khoản Docker Hub (đã verify email).
- Quyền mạng ra Internet để pull image/plugin.

Tùy chọn: Nếu chạy Jenkins cục bộ bằng Docker, chỉ cần Docker; không cần cài Jenkins native.

---

## 4) Khởi chạy Jenkins bằng Docker Compose

1) Vào thư mục dự án và chạy Jenkins:

```
sudo docker compose -f lesson11_CICD_jenkins/docker-compose.yaml up -d --build
```

- Compose sẽ build image Jenkins từ `lesson11_CICD_jenkins/custom_jenkins/` và cài sẵn plugin.
- Volume `jenkins_home` giữ cấu hình Jenkins.

2) Mở Jenkins: http://localhost:8081

3) Nếu cần làm sạch toàn bộ (xóa plugin hỏng, cấu hình cũ):

```
sudo docker compose -f lesson11_CICD_jenkins/docker-compose.yaml down -v
sudo docker compose -f lesson11_CICD_jenkins/docker-compose.yaml up -d --build
```

Lưu ý: `down -v` xóa cả dữ liệu Jenkins (mất cấu hình/cred cũ).

---

## 5) Cài đặt bắt buộc trong Jenkins

1) Plugins (đã nằm trong custom image):
- Docker Pipeline (`docker-workflow`)
- Pipeline: Declarative (`pipeline-model-definition`, `pipeline-model-api`, `pipeline-model-extensions`)
- Workflow Aggregator, Credentials, Git, JUnit, Timestamper, Matrix Auth…

Nếu bạn dùng Jenkins khác, hãy cài các plugin trên ở Manage Plugins.

2) Docker Hub Credentials:
- Tạo Access Token trên Docker Hub: Account → Security → New Access Token.
- Jenkins → Manage Jenkins → Credentials → System → Global → Add Credentials
  - Kind: Username with password
  - Username: tên Docker Hub (chữ thường)
  - Password: Access Token
  - ID: `dockerhub` (trùng với giá trị trong Jenkinsfile)

3) Tạo repository trên Docker Hub (thuộc tài khoản của bạn):
- Ví dụ: `youruser/house-price-prediction-api`.
- Sửa `registry` trong `Jenkinsfile` về repo của bạn nếu cần.

---

## 6) Jenkinsfile – biến và các stage

`Jenkinsfile` (ở gốc repo) định nghĩa các biến môi trường:

- `registry`: tên đầy đủ image trên Docker Hub, ví dụ `youruser/house-price-prediction-api`.
- `registryCredential`: ID credential Docker Hub trong Jenkins (mặc định `dockerhub`).
- `APP_DIR`: thư mục chứa code và Dockerfile (`lesson11_CICD_jenkins`).

Các stage chính:
- Test: chạy trong container `python:3.8`, cài requirements và `pytest` trong `APP_DIR`.
- Build: build Docker image bằng Dockerfile trong `APP_DIR`, tag `:<BUILD_NUMBER>` và `:latest`.
- Push: đăng nhập Docker Hub bằng `registryCredential` và push 2 tag.
- Deploy (placeholder): chỗ để bạn thêm logic triển khai (SSH, compose, k8s…).

File tham khảo: `Jenkinsfile`.

---

## 7) Thiết lập Job trong Jenkins

Bạn có 2 cách phổ biến:

A) Multibranch Pipeline (khuyến nghị):
- Tạo Multibranch Job, trỏ tới repo Git (HTTPS/SSH).
- Build Configuration: by Jenkinsfile (Script Path: `Jenkinsfile`).
- Scan sẽ tạo job cho từng nhánh có `Jenkinsfile`.

B) Pipeline (Single branch):
- Tạo Pipeline Job, chọn “Pipeline script from SCM”, chỉ nhánh cần build.

Webhook (tùy chọn): cấu hình GitHub webhook để Jenkins tự build khi push.

---

## 8) Chạy thử pipeline

1) Commit/push thay đổi lên nhánh Jenkins theo dõi (thường `main`).
2) Vào Jenkins job → Build Now (hoặc Scan Repository Now với Multibranch).
3) Theo dõi log cho 3 stage: Test → Build → Push. Khi thành công:
- Docker Hub repo của bạn sẽ có 2 tag mới: `:<BUILD_NUMBER>` và `:latest`.

---

## 9) Chạy ứng dụng đã build (local/server)

Pull và chạy image:

```
docker pull <youruser>/house-price-prediction-api:<BUILD_NUMBER>
docker run -d --name house-api -p 30000:30000 <youruser>/house-price-prediction-api:<BUILD_NUMBER>
```

Gọi thử API:

```
curl -X POST http://localhost:30000/predict \
  -H 'Content-Type: application/json' \
  -d '{
        "MSSubClass":60,
        "MSZoning":"RL",
        "LotArea":7844,
        "LotConfig":"Inside",
        "BldgType":"1Fam",
        "OverallCond":7,
        "YearBuilt":1978,
        "YearRemodAdd":1978,
        "Exterior1st":"HdBoard",
        "BsmtFinSF2":0.0,
        "TotalBsmtSF":672.0
      }'
```

Docker Compose (ví dụ trên server):

```yaml
services:
  api:
    image: <youruser>/house-price-prediction-api:<BUILD_NUMBER>
    ports:
      - "30000:30000"
```

---

## 10) Xử lý sự cố thường gặp

- authentication required – email must be verified:
  - Verify email Docker Hub → `docker logout` (cả user và `sudo`) → login lại.

- unauthorized: incorrect username or password (khi push):
  - Sai username hoặc dùng mật khẩu khi bật 2FA. Dùng Access Token.
  - Credential ID trong Jenkins phải đúng (`dockerhub`).
  - Repo Docker Hub phải thuộc user/organization bạn có quyền push.

- Jenkinsfile not found (Multibranch scan):
  - Đảm bảo có `Jenkinsfile` ở gốc repo, hoặc đặt Script Path đúng.

- Invalid agent type 'docker':
  - Thiếu plugins Docker Pipeline + Declarative. Cài plugin hoặc dùng image Jenkins custom.

- Hàng loạt plugin “X” hồng khi khởi động Jenkins:
  - Jenkins không tải được plugin online. Dùng image custom kèm `plugins.txt` hoặc kiểm tra proxy mạng.

- Cảnh báo Did you forget the `def` keyword:
  - Khai báo biến pipeline bằng `def` trước khi gán (ví dụ `def img = docker.build(...)`).

- Cảnh báo LegacyKeyValueFormat "ENV key value":
  - Trong Dockerfile app đổi `ENV KEY value` → `ENV KEY=value`.

- Làm sạch hoàn toàn Jenkins để rebuild plugin:
  - `down -v` rồi `up -d --build` với compose ở `lesson11_CICD_jenkins/docker-compose.yaml`.

---

## 11) Ghi chú bảo mật & thực hành tốt

- Không hardcode secret trong `Jenkinsfile`. Dùng Credentials của Jenkins.
- Dùng tag phiên bản bất biến (`:<BUILD_NUMBER>`) khi deploy để dễ rollback.
- Tách quyền: token Docker Hub chỉ cần quyền push cho repo cần thiết.
- Sao lưu volume `jenkins_home` định kỳ nếu đây là Jenkins lâu dài.

---

## 12) Lệnh nhanh thường dùng

- Khởi động Jenkins: `sudo docker compose -f lesson11_CICD_jenkins/docker-compose.yaml up -d --build`
- Xóa Jenkins + dữ liệu: `sudo docker compose -f lesson11_CICD_jenkins/docker-compose.yaml down -v`
- Build và chạy app local (không qua Jenkins):

```
docker build -t <youruser>/house-price-prediction-api:local -f lesson11_CICD_jenkins/Dockerfile lesson11_CICD_jenkins
docker run -d -p 30000:30000 --name house-api <youruser>/house-price-prediction-api:local
```

---

## 13) Tham chiếu file trong repo

- `Jenkinsfile`
- `lesson11_CICD_jenkins/Dockerfile`
- `lesson11_CICD_jenkins/main.py`
- `lesson11_CICD_jenkins/requirements.txt`
- `lesson11_CICD_jenkins/custom_jenkins/Dockerfile`
- `lesson11_CICD_jenkins/custom_jenkins/plugins.txt`
- `lesson11_CICD_jenkins/docker-compose.yaml`

Chúc bạn học và thực hành Jenkins hiệu quả!

