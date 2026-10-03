# BÀI THỰC HÀNH JENKINS CI/CD - DEVOPS
**Họ và tên / Sinh viên**: tiepnie (thanh105555-source / devops-26-8)  
**File báo cáo**: `devops-jenkins-ytiepnie.md`

---

## 1. Mục tiêu bài lab
Dựng Jenkins CI/CD ngay trên laptop (Local Docker), thực hiện đầy đủ 5 bước CI/CD:
`BUILD → TEST → PUSH → PULL → DEPLOY`

kết thúc bằng ứng dụng **Tra từ điển Anh - Việt** chạy thực tế ở `http://localhost:3000` từ image kéo về từ registry `ghcr.io/thanh105555-source/devops-26-8`.

---

## 2. Kiến trúc tổng quan (DooD - Docker outside of Docker)

```
   ┌─────────────── LAPTOP (Local Environment) ───────────────┐
   │                                                          │
   │   ┌── container jenkins ──┐                              │
   │   │  Jenkins + docker CLI │                              │
   │   │                       │                              │
   │   │  docker build ────────┼──┐                           │
   │   │  docker compose up ───┼──┤                           │
   │   └───────────────────────┘  │                           │
   │              │               │ qua /var/run/docker.sock  │
   │              │               ▼                           │
   │              │      ┌─── Docker Desktop ───┐             │
   │              │      │  (daemon thật)       │             │
   │              │      │                      │             │
   │              │      │  dictionary-prod-web ├──► :3000    │
   │              │      │  dictionary-prod-db  ├──► :5432    │
   │              │      └──────────────────────┘             │
   └──────────────┼───────────────────────────────────────────┘
                  │
                  ▼ push / pull
         ghcr.io/thanh105555-source/devops-26-8
```

---

## 3. Các bước triển khai

### Step 1: Tạo Token GHCR trên GitHub
1. GitHub → Settings → Developer Settings → Personal Access Tokens (Classic).
2. Quyền: `write:packages`, `read:packages`.
3. Tạo token: `jenkins-ghcr`.

### Step 2: Cấu hình `Jenkinsfile`
Trong file `Jenkinsfile`, khai báo `IMAGE_NAME`:
```groovy
IMAGE_NAME = 'thanh105555-source/devops-26-8'
```

### Step 3: Dựng Jenkins Container
```bash
cd jenkins
docker compose up -d --build
```
Truy cập: `http://localhost:8080`.

### Step 4: Tạo Credential trong Jenkins
- Manage Jenkins → Credentials → Global credentials → Add Credentials.
- Kind: `Username with password`
- ID: `ghcr-credentials`
- Username: `tiepnie`
- Password: `<GHCR_TOKEN>`

### Step 5: Cấu hình Pipeline Job
- Name: `dictionary-cicd`
- Definition: `Pipeline script from SCM`
- SCM: `Git`
- Repository URL: `https://github.com/thanh105555-source/devops-26-8.git`
- Credentials: `ghcr-credentials`
- Branch: `*/main`
- Script Path: `Jenkinsfile`

---

## 4. Các Stage trong Jenkins Pipeline

1. **Chuẩn bị**: Lấy git commit SHA, đăng nhập GHCR registry, tạo builder buildx multi-arch.
2. **1. BUILD**: Build image native cho môi trường test local.
3. **2. TEST**: Chạy `smoke-test.sh` kiểm tra 8 test cases API & DB kết nối.
4. **3. PUSH**: Build đa kiến trúc (`linux/amd64,linux/arm64`) và push lên `ghcr.io/thanh105555-source/devops-26-8`.
5. **4. PULL**: Xóa image local cũ (`docker rmi`) và kéo (`docker pull`) image mới từ GHCR về để đảm bảo registry hoạt động tốt.
6. **5. DEPLOY**: Khởi chạy stack bằng `docker-compose.prod.yml` ra cổng `3000`.
7. **6. Kiểm tra sau deploy**: Gọi API healthcheck `http://host.docker.internal:3000/api/health` xác nhận app hoạt động.

---

## 5. Kiểm tra kết quả thành công
- Web Tra từ điển Anh - Việt hoạt động tại: **`http://localhost:3000`**
- Database PostgreSQL đếm 10 từ ban đầu kết nối thành công.
