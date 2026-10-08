
| Lệnh | Ý nghĩa & Giải thích tham số |
| :--- | :--- |
| `git init` | Khởi tạo một kho chứa Git (local repository) rỗng tại thư mục hiện tại. |
| `git add .` | Đưa tất cả các file vừa tạo/chỉnh sửa vào khu vực chuẩn bị commit (Staging Area). Dấu `.` đại diện cho toàn bộ thư mục hiện tại. |
| `git commit -m "feat: initial static site"` | Đóng gói các thay đổi vào lịch sử Git với thông điệp mô tả (`-m` viết tắt của message). |
| `git branch -M main` | Đổi tên nhánh hiện tại thành `main` (`-M` là ép buộc đổi tên nhánh chính theo chuẩn mới của GitHub). |
| `git remote add origin <URL_REPO>` | Liên kết kho lưu trữ trên máy tính với repository từ xa trên GitHub đặt tên là `origin`. |
| `git push -u origin main` | Đẩy commit lên nhánh `main` của `origin`. Tham số `-u` (upstream) giúp các lần sau chỉ cần gõ `git push`. |


 Thiết lập User an toàn trên máy chủ Linux

Đăng nhập ban đầu bằng tài khoản `root`:
```bash
ssh root@<IP_SERVER>
```

Tiến hành tạo user quản trị riêng để tránh thao tác trực tiếp bằng `root`:

| Lệnh | Ý nghĩa & Giải thích tham số |
| :--- | :--- |
| `apt update && apt upgrade -y` | `apt update` làm mới danh sách package từ repository; `apt upgrade -y` cập nhật các phần mềm lên bản mới nhất mà không cần hỏi xác nhận (`-y`). |
| `adduser deployer` | Tạo một tài khoản người dùng mới tên là `deployer` cùng với thư mục Home (`/home/deployer`). |
| `usermod -aG sudo deployer` | Thêm user `deployer` vào nhóm `sudo`. Tham số `-a` (append - thêm vào) và `-G` (nhóm phụ) giúp user có quyền chạy lệnh quản trị với `sudo`. |
| `su - deployer` | Chuyển đổi phiên làm việc sang user `deployer` (dấu `-` để tải toàn bộ biến môi trường của user đó). |

---

### Bước 3: Cấu hình tường lửa UFW (Firewall tầng hệ điều hành)
| Lệnh | Ý nghĩa & Giải thích tham số |
| :--- | :--- |
| `sudo ufw allow 22/tcp` | Cho phép kết nối vào cổng 22 (SSH) giao thức TCP. Có thể dùng `sudo ufw allow OpenSSH`. |
| `sudo ufw allow 80/tcp` | Cho phép kết nối vào cổng 80 (HTTP) để người dùng truy cập web. |
| `sudo ufw allow 443/tcp` | Cho phép kết nối vào cổng 443 (HTTPS) khi cài chứng chỉ SSL sau này. |
| `sudo ufw enable` | Bật tường lửa UFW và tự động khởi động cùng hệ thống. |
| `sudo ufw status verbose` | Kiểm tra chi tiết trạng thái UFW và danh sách các quy tắc (Rule) đang có hiệu lực. |

-

### Bước 4: Mở cổng trên Tường lửa Cloud / VPS (WiServices, AWS, v.v.)

Tường lửa hệ điều hành (UFW) chỉ là lớp bên trong. Các nhà cung cấp Cloud luôn có thêm một lớp tường lửa bên ngoài (Security Group):

1. Vào Dashboard máy ảo > **Tường lửa & Quy tắc** > Chọn **Thêm Quy Tắc**.
2. Thiết lập quy tắc cho cổng Web:
    - **Hướng:** `Đầu Vào (IN)`
    - **Hành Động:** `ACCEPT`
    - **Giao Thức:** `TCP`
    - **NGUỒN (phía gửi):** IP: `All` | Cổng: `Để trống` hoặc `All`
    - **ĐÍCH (phía nhận):** IP: `All` | Cổng: **`80`**
3. Gạt công tắc **Bật Quy Tắc** > Nhấn **Lưu Thay Đổi**.

---

### Bước 5: Cài đặt và cấu hình Nginx

| Lệnh | Ý nghĩa & Giải thích tham số |
| :--- | :--- |
| `sudo apt install nginx git -y` | Cài đặt máy chủ web Nginx và công cụ Git. |
| `sudo systemctl enable nginx` | Cấu hình cho Nginx tự động chạy mỗi khi máy chủ khởi động lại. |
| `sudo systemctl start nginx` | Khởi chạy dịch vụ Nginx ngay lập tức. |
| `sudo systemctl status nginx` | Kiểm tra trạng thái hoạt động của tiến trình Nginx. |
| `sudo mkdir -p /var/www/my-portfolio` | Tạo thư mục chứa mã nguồn web. Tham số `-p` (parents) tự động tạo các thư mục cha nếu chưa có. |

#### Tạo file cấu hình Virtual Host cho Nginx:
Chạy lệnh:
```bash
sudo nano /etc/nginx/sites-available/my-portfolio
```

Nội dung cấu hình:
```nginx
server {
    listen 80;
    listen [::]:80;

    # Điền IP hoặc domain của bạn (dùng _ để chấp nhận mọi IP/domain)
    server_name 221.121.3.247 _;

    # Thư mục gốc chứa file index.html
    root /var/www/my-portfolio;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/my-portfolio.access.log;
    error_log /var/log/nginx/my-portfolio.error.log;
}

1. **Đảm bảo thư mục web đã tồn tại:**
   ```bash
   sudo mkdir -p /var/www/my-portfolio
   ```

2. **Tạo nội dung file `index.html` (chọn 1 trong các cách sau):**

    - **Cách 1: Dùng lệnh `cat` và `tee` để ghi nhanh toàn bộ nội dung (Khuyên dùng):**
      ```bash
      cat << 'EOF' | sudo tee /var/www/my-portfolio/index.html
      