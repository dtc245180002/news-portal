# ĐỀ 19: HỆ THỐNG WEBSITE TIN TỨC / CỔNG THÔNG TIN (WORDPRESS)
**Môn học:** Triển khai và Quản trị Hệ thống Phần mềm

## 1. Thông tin sinh viên thực hiện
- **Họ và tên:** [Điền Họ Tên Của Bạn]
- **Mã số sinh viên (MSSV):** [Điền MSSV Của Bạn]
- **Lớp:** [Điền Lớp Của Bạn]
- **Đề tài:** Đề 19 - Website Tin tức / Cổng thông tin (Sử dụng WordPress + MySQL + phpMyAdmin + Nginx + Prometheus + Grafana + Loki)

---

## 2. Kiến trúc & Cấu trúc thư mục dự án
Hệ thống được đóng gói hoàn toàn bằng **Docker Compose** và phân tách thành 3 mạng nội bộ (`frontend_net`, `backend_net`, `monitoring_net`):

```text
de19-news-portal/
├── docker-compose.yml          # Khai báo toàn bộ 9 services & 3 networks cô lập
├── .env.example                # File mẫu cấu hình biến môi trường & mật khẩu mạnh
├── .gitignore                  # Ngăn đẩy file nhạy cảm (.env, dữ liệu DB) lên GitHub
├── mysql-init/
│   └── init.sql                # Script tạo user exporter với quyền tối thiểu (Least Privilege)
├── nginx/
│   ├── nginx.conf              # Cấu hình Reverse Proxy, HTTPS (TLS) & Security Headers
│   └── certs/                  # Chứa chứng chỉ SSL tự ký (selfsigned.crt & selfsigned.key)
├── prometheus/
│   ├── prometheus.yml          # Cấu hình thu thập metrics (cAdvisor, Nginx, MySQL)
│   └── .my.cnf                 # Thông tin kết nối an toàn cho MySQLd Exporter
├── loki/
│   └── loki-config.yml         # Cấu hình máy chủ lưu trữ log tập trung Loki
├── promtail/
│   └── promtail-config.yml     # Cấu hình thu thập log từ Docker containers & Nginx
└── README.md                   # Tài liệu hướng dẫn triển khai và vận hành hệ thống
```

---

## 3. Hướng dẫn cài đặt và khởi chạy hệ thống

### Bước 1: Clone repository từ GitHub
```bash
git clone [https://github.com/](https://github.com/)<TEN_TAI_KHOAN_MSSV>/<TEN_REPO>.git
cd <TEN_REPO>
```

### Bước 2: Cấu hình biến môi trường (.env)
Sao chép file biến môi trường mẫu và thiết lập mật khẩu mạnh:
```bash
cp .env.example .env
```

### Bước 3: Khởi tạo chứng chỉ SSL tự ký (HTTPS) cho Nginx
```bash
mkdir -p nginx/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/selfsigned.key \
  -out nginx/certs/selfsigned.crt \
  -subj "/C=VN/ST=ThaiNguyen/L=ThaiNguyen/O=ICTU/OU=CNTT/CN=192.168.182.129"
```

### Bước 4: Khởi chạy toàn bộ dịch vụ bằng Docker Compose
```bash
docker-compose up -d
```
Kiểm tra trạng thái hoạt động của các container:
```bash
docker-compose ps
```

---

## 4. Danh sách cổng dịch vụ (Endpoints) & Thông tin truy cập

| Dịch vụ | Container Name | Địa chỉ truy cập (URL) | Thông tin đăng nhập |
| :--- | :--- | :--- | :--- |
| **Website Tin tức (Qua Nginx HTTPS)** | `news_nginx` / `news_wordpress` | `https://192.168.182.129` | Trang độc giả |
| **Trang Quản trị đăng bài (WP-Admin)** | `news_wordpress` | `https://192.168.182.129/wp-admin` | Tài khoản Admin WordPress |
| **Quản trị CSDL (phpMyAdmin)** | `news_phpmyadmin` | `http://192.168.182.129:8081` | User: `wp_news_user` (xem `.env`) |
| **Prometheus Monitoring** | `news_prometheus` | `http://192.168.182.129:9090/targets` | Không yêu cầu |
| **Grafana Dashboard & LogQL** | `news_grafana` | `http://192.168.182.129:3000` | `admin` / `Grafana@Admin2026!` |
| **Loki Log Server API** | `news_loki` | `http://192.168.182.129:3100` | Kết nối nội bộ từ Grafana |

---

## 5. Giám sát (Prometheus + Grafana) & Truy vấn Log (LogQL)

### 5.1. Các Grafana Dashboard sử dụng (Data source: `http://prometheus:9090`)
- **Giám sát Docker Containers (cAdvisor):** Import Dashboard ID `14282` (hoặc `193`)
- **Giám sát Web Server (Nginx Exporter):** Import Dashboard ID `12708`
- **Giám sát Database (MySQLd Exporter):** Import Dashboard ID `7362`

### 5.2. Các câu truy vấn LogQL mẫu trên Grafana Explore (Data source: `http://loki:3100`)
- **Query 1 - Xem toàn bộ log truy cập và phản hồi qua Nginx Reverse Proxy:**
  ```logql
  {container="news_nginx"}
  ```
- **Query 2 - Lọc các hành vi truy cập trang quản trị đăng bài hoặc đăng nhập Admin:**
  ```logql
  {container="news_wordpress"} |= "wp-admin" or {container="news_wordpress"} |= "wp-login.php"
  ```
- **Query 3 - Thống kê tốc độ sinh log theo từng container trong 5 phút:**
  ```logql
  sum by (container) (rate({container=~"news_.*"}[5m]))
  ```

---

## 6. Các biện pháp Bảo mật (Hardening) đã áp dụng
1. **Non-root Containers & Chặn leo thang đặc quyền:** Các dịch vụ (`prometheus`, `grafana`, `loki`, `nginx-exporter`, `mysql-exporter`) chạy với UID non-root (`65534`, `472`, `10001`) kèm cấu hình `security_opt: ["no-new-privileges:true"]`.
2. **Network Isolation (Cô lập mạng):** Tách biệt thành 3 mạng `frontend_net`, `backend_net`, `monitoring_net`. Container `nginx` không thể kết nối trực tiếp tới `db` (MySQL).
3. **Bảo mật mật khẩu mạnh (.env):** Sử dụng mật khẩu phức tạp trong file `.env` và loại trừ khỏi Git qua `.gitignore`.
4. **Hạn chế quyền Database (Least Privilege):** Tách biệt user ứng dụng (`wp_news_user`) và user giám sát (`exporter` chỉ có quyền `SELECT, PROCESS, REPLICATION CLIENT`, giới hạn tối đa 3 kết nối).
5. **Nginx HTTPS & Security Headers:** Sử dụng SSL/TLS tự ký, ẩn phiên bản (`server_tokens off;`) và bật đầy đủ các header bảo mật (`X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, `X-XSS-Protection: 1; mode=block`, `Strict-Transport-Security`).
