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

4) GitHub Credentials (nếu repo private hoặc để tránh rate limit):
- Tạo Personal Access Token (PAT) trên GitHub:
  - GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token.
  - Scopes tối thiểu: `repo` (đọc mã nguồn). Nếu cần webhook từ Jenkins, thêm `admin:repo_hook`. Với repo/organization riêng tư, có thể cần `read:org`.
  - Chọn thời hạn phù hợp, tạo token và sao chép (chỉ hiện 1 lần).
- Thêm token vào Jenkins:
  - Manage Jenkins → Credentials → System → Global → Add Credentials.
  - Cách 1 (thường dùng cho Git over HTTPS): Kind: `Username with password` → Username: tài khoản GitHub của bạn → Password: dán PAT → ID: `github-https`.
  - Cách 2 (dùng cho GitHub API): Kind: `Secret text` → Secret: PAT → ID: `github-token`.
- Gán credentials cho job:
  - Multibranch Pipeline → Configure → Branch Sources → Add source (GitHub/Git) → nhập URL repo.
  - Chọn `Credentials`: `github-https` (nếu dùng HTTPS). Với nguồn “GitHub” bạn cũng có thể chọn `github-token` cho phần "Scan credentials" để giảm rate limit.
- Tùy chọn cấu hình GitHub API usage:
  - Manage Jenkins → Configure System → GitHub → GitHub API usage → chọn chiến lược dùng token (ví dụ Exhaust/Throttle) và trỏ tới credential `github-token` để tránh thông báo "GitHub throttling is disabled".

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

---

## Phụ lục: Dùng Ngrok cho Jenkins local (cổng 8081)

**Vai trò**
- Ngrok tạo đường hầm (secure tunnel) để công khai Jenkins đang chạy trên máy local (port 8081) ra Internet. Nhờ đó GitHub có thể gửi Webhook vào Jenkins của bạn để kích hoạt build tự động, và bạn cũng có thể truy cập Jenkins từ bên ngoài khi demo.

**Cài đặt và chạy**
- Cài ngrok: `brew install ngrok` (macOS) hoặc tải từ https://ngrok.com/download.
- Đăng ký tài khoản và thêm authtoken: `ngrok config add-authtoken <YOUR_TOKEN>`.
- Mở tunnel cho Jenkins: `ngrok http 8081`
  - Ghi lại URL HTTPS có dạng `https://<subdomain>.ngrok.app`.

**Cấu hình Jenkins và GitHub webhook**
- Jenkins URL: Manage Jenkins → Configure System → Jenkins URL → đặt `https://<subdomain>.ngrok.app/`.
- GitHub Webhook (trên repo): Settings → Webhooks → Add webhook
  - Payload URL: `https://<subdomain>.ngrok.app/github-webhook/` (lưu ý dấu “/” cuối).
  - Content type: `application/json`.
  - Secret: tùy chọn. Có thể đặt và cấu hình tương ứng trong GitHub plugin/credentials của Jenkins để tăng bảo mật.
  - Events: chọn `Just the push event` và nếu dùng Multibranch, tick thêm `Pull requests`.
- Lưu. Thử push code → xem Jenkins có build tự động không. Bạn cũng có thể mở dashboard request của ngrok để thấy POST từ GitHub.

**Lưu ý quan trọng**
- URL ngrok free sẽ thay đổi mỗi lần chạy: mỗi lần bạn `ngrok http 8081`, cần cập nhật lại Jenkins URL và GitHub webhook. Muốn cố định URL, dùng ngrok bản trả phí (reserved domain) hoặc triển khai Jenkins trên máy chủ công khai.
- Bảo mật: Khi công khai Jenkins ra Internet, bảo vệ tài khoản admin, bật phân quyền (Matrix-based security), hạn chế chia sẻ URL. Tránh bật basic-auth trực tiếp trên ngrok nếu bạn cần nhận webhook GitHub (GitHub không gửi kèm basic-auth), trừ khi bạn có cấu hình loại trừ endpoint webhook.
- Nếu thấy lỗi liên quan proxy/URL không khớp, đảm bảo Jenkins URL dùng `https` (trùng với ngrok) để tránh vấn đề CSRF/crumb và chuyển hướng.

---

## 14) Cấu trúc dự án và giải thích chi tiết

Mục này giúp bạn hiểu rõ từng thư mục, từng file và các dòng lệnh cốt lõi để có thể tự viết lại dự án tương tự.

### 14.1 Thư mục gốc

- `Jenkinsfile`: Định nghĩa pipeline CI/CD cho toàn bộ repo.

Giải thích từng phần trong `Jenkinsfile`:

```groovy
pipeline {
    agent any
```
- Khai báo dùng bất kỳ agent nào (node/agent Jenkins) để chạy pipeline.

```groovy
    options {
        buildDiscarder(logRotator(numToKeepStr: '5', daysToKeepStr: '5'))
        timestamps()
    }
```
- `buildDiscarder`: giữ tối đa 5 bản build hoặc 5 ngày (cái nào đến trước) để tiết kiệm dung lượng.
- `timestamps()`: thêm timestamp vào log cho dễ theo dõi.

```groovy
    environment {
        registry = '<youruser>/house-price-prediction-api'
        registryCredential = 'dockerhub'
        APP_DIR = 'lesson11_CICD_jenkins'
    }
```
- `registry`: tên full của Docker image sẽ push lên Docker Hub.
- `registryCredential`: ID credential Docker Hub trong Jenkins.
- `APP_DIR`: thư mục con chứa app, Dockerfile, requirements, tests.

```groovy
    stages {
        stage('Test') {
            agent { docker { image 'python:3.8' } }
            steps {
                echo 'Testing model correctness..'
                dir(env.APP_DIR) {
                    sh 'pip install -r requirements.txt && pytest'
                }
            }
        }
```
- Stage Test chạy trong container `python:3.8`, đảm bảo môi trường sạch.
- `dir(APP_DIR)`: cd vào thư mục app rồi cài `requirements.txt` và chạy `pytest`.

```groovy
        stage('Build') {
            steps {
                script {
                    echo 'Building image for deployment..'
                    def imageTag = "${registry}:${env.BUILD_NUMBER}"
                    dockerImage = docker.build(imageTag, "-f ${env.APP_DIR}/Dockerfile ${env.APP_DIR}")
                    echo 'Pushing image to dockerhub..'
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }
```
- Tính tag version theo số build của Jenkins (`BUILD_NUMBER`).
- Build image dùng Dockerfile trong `APP_DIR` (context cũng là `APP_DIR`).
- Đăng nhập registry bằng credential rồi push 2 tag: bản có số build (bất biến) và `latest` (di động).

```groovy
        stage('Deploy') {
            steps {
                echo 'Deploying models..'
                echo 'Trigger remote deployment script or compose here.'
            }
        }
    }
}
```
- Placeholder cho bước triển khai; có thể thêm SSH tới server, chạy compose/k8s.

### 14.2 Thư mục `lesson11_CICD_jenkins/`

Chứa mã nguồn ứng dụng, Dockerfile để build image, test, và cấu hình Jenkins docker-compose.

#### a) `Dockerfile` (ứng dụng FastAPI)

```dockerfile
FROM python:3.8
```
- Dùng image nền Python 3.8 chính thức.

```dockerfile
WORKDIR /app
```
- Đặt thư mục làm việc mặc định là `/app`.

```dockerfile
COPY ./main.py /app
COPY ./requirements.txt /app
COPY ./models /app/models
```
- Sao chép file mã nguồn, requirements và thư mục model vào image.

```dockerfile
ENV MODEL_PATH /app/models/model.pkl
```
- Biến môi trường chỉ đường dẫn đến model (có thể override khi chạy container).

```dockerfile
EXPOSE 30000
```
- Tài liệu hóa port dịch vụ (không tự mở cổng; cần `-p` khi `docker run`).

```dockerfile
RUN pip install -r requirements.txt --no-cache-dir
```
- Cài dependencies, tắt cache để giảm kích thước image.

```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "30000"]
```
- Lệnh chạy app khi container khởi động.

#### b) `main.py` (API FastAPI)

Các điểm chính:
- Import `joblib`, `pandas`, `FastAPI`, `BaseModel` để load model và định nghĩa schema.
- Tạo app: `app = FastAPI()`.
- Khai báo lớp `HouseInfo(BaseModel)` với các field và giá trị mặc định — đây là payload cho endpoint.
- Load model: `joblib.load(os.environ.get('MODEL_PATH', 'models/model.pkl'))` — ưu tiên biến môi trường.
- Endpoint `@app.post('/predict')`:
  - Log "Make predictions..." bằng `loguru`.
  - Chuyển payload thành `DataFrame` (dùng `jsonable_encoder` để chắc chắn serializable).
  - `clf.predict(...)[0]` lấy giá trị dự đoán đầu tiên và trả về JSON `{ 'price': ... }`.

#### c) `requirements.txt`

- `fastapi`, `uvicorn[standard]`: framework API và web server.
- `loguru`: logging tiện dụng.
- `joblib`, `scikit-learn`, `pandas`, `lightgbm`: phục vụ load và chạy mô hình ML.
- `pytest`: chạy test tự động trong pipeline.

#### d) `tests/test_model_correctness.py`

- Load model từ `models/model.pkl`.
- Tạo một dict dữ liệu mẫu, chuyển thành `DataFrame`.
- `assert` giá trị dự đoán đúng kỳ vọng → đảm bảo model và mã dự đoán hoạt động ổn định.

#### e) `client.py`

- Chuẩn bị headers và JSON payload giống `HouseInfo`.
- Gửi `requests.post` tới `http://localhost:30001/predict` (sửa port nếu chạy khác).
- In kết quả/ lỗi; hữu ích để kiểm tra thủ công ngoài `pytest`.

#### f) `docker-compose.yaml` (chạy Jenkins trong Docker)

```yaml
version: '3.8'
services:
  jenkins:
    build: ./custom_jenkins
    image: local/custom-jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    privileged: true
    user: root
    ports:
      - 8081:8080
      - 50000:50000
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
```
- Build image Jenkins tùy biến để cài sẵn Docker CLI và plugins.
- Mở cổng 8081 (UI Jenkins) và 50000 (JNLP agent).
- Mount socket Docker để pipeline có thể gọi `docker build/push`.
- Volume `jenkins_home` lưu cấu hình và dữ liệu Jenkins.

```yaml
volumes:
  jenkins_home:
```
- Khai báo volume có tên, tái sử dụng giữa các lần khởi động.

### 14.3 `lesson11_CICD_jenkins/custom_jenkins/`

#### a) `Dockerfile` (Jenkins custom)

```dockerfile
FROM jenkins/jenkins:lts
```
- Image Jenkins LTS chính thức.

```dockerfile
USER root
RUN curl -fsSL https://get.docker.com > /tmp/dockerinstall \
    && chmod +x /tmp/dockerinstall \
    && /tmp/dockerinstall \
    && rm -f /tmp/dockerinstall
```
- Cài Docker CLI trong container Jenkins để pipeline có lệnh `docker`.

```dockerfile
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
```
- Cài sẵn các plugin cần thiết, tránh phụ thuộc mạng lúc khởi động.

```dockerfile
USER jenkins
```
- Trả quyền chạy về user `jenkins` (bảo mật tốt hơn).

#### b) `plugins.txt`

- `workflow-aggregator`: tập hợp plugin core cho Pipeline.
- `docker-workflow`: cung cấp steps Docker cho pipeline (`docker.build`, `withRegistry`, ...).
- `pipeline-model-definition/api/extensions`: hỗ trợ Declarative Pipeline (cần để dùng `agent { docker ... }`).
- `credentials`, `ssh-credentials`, `credentials-binding`: quản lý và bind secret/credential vào stage.
- `git`, `git-client`: tích hợp Git.
- `timestamper`, `junit`, `mailer`, `matrix-auth`: tiện ích log/junit/email/phân quyền.

### 14.4 Các thư mục khác

- `models/`: chứa `model.pkl` được `joblib.load` khi khởi động app.
- `data/`, `notebooks/`: dữ liệu và notebook minh họa (không tham gia vào build runtime, chỉ để tham khảo/học tập).

Với phần giải thích trên, bạn có thể khởi tạo lại dự án từ đầu: tạo API FastAPI, viết Dockerfile, thêm test với pytest, viết Jenkinsfile (Test → Build → Push), dựng Jenkins bằng docker-compose, cấu hình credentials và chạy pipeline.
