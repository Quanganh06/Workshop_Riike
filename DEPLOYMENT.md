# Hướng dẫn Deploy lên AWS EC2

## 📋 Checklist Deployment

### Bước 1: Chuẩn bị Docker Hub

1. **Tạo tài khoản Docker Hub** (nếu chưa có)
   - Truy cập: https://hub.docker.com
   - Đăng ký tài khoản miễn phí

2. **Tạo Access Token**
   - Vào Docker Hub → Account Settings → Security → New Access Token
   - Tạo token với quyền Read & Write
   - **Lưu lại token này** (sẽ dùng cho GitHub Secrets)

3. **Cập nhật Docker image name trong CI/CD**
   - Mở file `.github/workflows/ci-cd.yaml`
   - Thay `your-dockerhub-username` bằng Docker Hub username của bạn (dòng 14)

---

### Bước 2: Tạo và Cấu hình EC2 Instance

1. **Tạo EC2 Instance**
   - Vào AWS Console → EC2 → Launch Instance
   - Chọn AMI: **Amazon Linux 2023** hoặc **Ubuntu 22.04 LTS**
   - Instance type: **t2.micro** (free tier) hoặc **t3.small**
   - Key pair: Tạo mới hoặc chọn key pair có sẵn
   - Security Group: Mở các port:
     - **22** (SSH)
     - **8080** (Application)
   - Launch instance

2. **Lấy thông tin EC2**
   - Public IP hoặc Public DNS của EC2 instance
   - Username (thường là `ec2-user` cho Amazon Linux, `ubuntu` cho Ubuntu)

3. **Tạo SSH Key Pair** (nếu chưa có)
   ```bash
   ssh-keygen -t rsa -b 4096 -f ~/.ssh/ec2-deploy-key
   ```
   - Copy public key vào EC2 instance
   - **Lưu private key** để dùng cho GitHub Secrets

---

### Bước 3: Setup EC2 Server

1. **Kết nối SSH vào EC2**
   ```bash
   ssh -i ~/.ssh/your-key.pem ec2-user@your-ec2-ip
   ```

2. **Cài đặt Docker trên EC2**
   
   **Cho Amazon Linux 2023:**
   ```bash
   sudo yum update -y
   sudo yum install docker -y
   sudo systemctl start docker
   sudo systemctl enable docker
   sudo usermod -aG docker ec2-user
   ```
   
   **Cho Ubuntu:**
   ```bash
   sudo apt-get update
   sudo apt-get install docker.io -y
   sudo systemctl start docker
   sudo systemctl enable docker
   sudo usermod -aG docker ubuntu
   ```

3. **Tạo thư mục ứng dụng**
   ```bash
   mkdir -p /home/ec2-user/app
   cd /home/ec2-user/app
   ```

4. **Tạo file `.env.prod` trên EC2**
   ```bash
   nano /home/ec2-user/app/.env.prod
   ```
   
   Nội dung file:
   ```bash
  
   ```
   
   **Lưu ý:** Điền đúng thông tin Aiven MySQL của bạn

5. **Cấp quyền cho file**
   ```bash
   chmod 600 /home/ec2-user/app/.env.prod
   ```

---

### Bước 4: Cấu hình GitHub Secrets

1. **Vào GitHub Repository**
   - Settings → Secrets and variables → Actions → New repository secret

2. **Thêm các Secrets sau:**

   | Secret Name | Giá trị | Mô tả |
   |------------|---------|-------|
   | `DOCKERHUB_USERNAME` | `your-dockerhub-username` | Docker Hub username |
   | `DOCKERHUB_TOKEN` | `your-dockerhub-token` | Docker Hub access token |
   | `EC2_HOST` | `your-ec2-public-ip` hoặc `your-ec2-dns` | EC2 public IP hoặc DNS |
   | `EC2_USERNAME` | `ec2-user` hoặc `ubuntu` | EC2 username |
   | `EC2_SSH_KEY` | Nội dung file private key | SSH private key để kết nối EC2 |

3. **Lấy SSH Private Key:**
   ```bash
   cat ~/.ssh/your-ec2-key.pem
   ```
   Copy toàn bộ nội dung (bao gồm `-----BEGIN RSA PRIVATE KEY-----` và `-----END RSA PRIVATE KEY-----`)

---

### Bước 5: Chạy SQL Script trên Aiven MySQL

1. **Kết nối Aiven MySQL**
   - Vào Aiven Console → Service → Connection Information
   - Copy connection string

2. **Chạy SQL script**
   ```bash
   mysql -h mysql-23d0d8d8-lequanganhh1624-3182.j.aivencloud.com -P 28932 -u avnadmin -p defaultdb < src/main/resources/db/re_shopping_cart_db.sql
   ```
   
   Hoặc dùng MySQL client/GUI tool để chạy script `src/main/resources/db/re_shopping_cart_db.sql`

---

### Bước 6: Cập nhật CI/CD Workflow

1. **Sửa Docker image name**
   - Mở `.github/workflows/ci-cd.yaml`
   - Dòng 14: Thay `your-dockerhub-username` bằng Docker Hub username thực tế

---

### Bước 7: Deploy

1. **Commit và Push code lên GitHub**
   ```bash
   git add .
   git commit -m "Setup deployment configuration"
   git push origin main
   ```

2. **Kiểm tra GitHub Actions**
   - Vào GitHub Repository → Actions tab
   - Xem workflow đang chạy
   - Đợi các jobs hoàn thành:
     - ✅ Build
     - ✅ Docker (build & push image)
     - ✅ Deploy (deploy lên EC2)

3. **Kiểm tra ứng dụng**
   - Truy cập: `http://your-ec2-ip:8080`
   - Ứng dụng sẽ tự động chạy trong Docker container

---

### Bước 8: Cấu hình Domain (Optional)

1. **Mua domain** (nếu cần)
2. **Cấu hình Route 53** hoặc DNS provider
3. **Point domain về EC2 IP**
4. **Cấu hình Nginx reverse proxy** (nếu muốn dùng port 80/443)

---

## 🔧 Troubleshooting

### Lỗi SSH Connection
- Kiểm tra Security Group có mở port 22
- Kiểm tra SSH key có đúng
- Kiểm tra EC2 username (`ec2-user` cho Amazon Linux, `ubuntu` cho Ubuntu)

### Lỗi Docker Connection
- Kiểm tra Docker đã được cài đặt trên EC2
- Kiểm tra user có trong docker group: `groups`
- Logout và login lại sau khi thêm user vào docker group

### Lỗi Database Connection
- Kiểm tra Aiven MySQL IP whitelist có cho phép EC2 IP
- Kiểm tra `.env.prod` trên EC2 có đúng thông tin database
- Kiểm tra Security Group của Aiven MySQL

### Lỗi Port đã được sử dụng
```bash
# Kiểm tra port 8080
sudo netstat -tulpn | grep 8080

# Kill process nếu cần
sudo kill -9 <PID>
```

---

## 📝 Notes

- **Security:** Đảm bảo `.env.prod` không được commit lên Git
- **Backup:** Nên backup database trước khi deploy
- **Monitoring:** Có thể setup CloudWatch để monitor EC2
- **SSL:** Có thể setup Let's Encrypt SSL certificate nếu cần HTTPS

---

## ✅ Checklist Summary

- [ ] Docker Hub account & token
- [ ] EC2 instance created
- [ ] Docker installed on EC2
- [ ] `.env.prod` file created on EC2
- [ ] GitHub Secrets configured
- [ ] SQL script executed on Aiven MySQL
- [ ] CI/CD workflow updated with Docker Hub username
- [ ] Code pushed to GitHub
- [ ] Application accessible at `http://ec2-ip:8080`

