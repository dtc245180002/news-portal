# HỆ THỐNG WEBSITE TIN TỨC / CỔNG THÔNG TIN (WORDPRESS)
**Môn học:** Triển khai và Quản trị Hệ thống Phần mềm

## 1. Thông tin sinh viên thực hiện
- **Họ và tên:** Lê Hồng Phong
- **Mã số sinh viên (MSSV):** DTC245180002
- **Lớp:** KTPM-K23C
- **Đề tài:** Đề 19 - Website Tin tức / Cổng thông tin (WordPress + MySQL + phpMyAdmin + Nginx + Prometheus + Grafana + Loki)

---

## 2. Kiến trúc & Cấu trúc thư mục dự án
Hệ thống được đóng gói hoàn toàn bằng **Docker Compose** với **11 dịch vụ (services)** và phân tách thành 3 mạng nội bộ (`frontend_net`, `backend_net`, `monitoring_net`):

```text
news-portal/
├── docker-compose.yml          # Khai báo toàn bộ 11 services & 3 networks cô lập
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
git clone https://github.com/dtc245180002/news-portal.git
cd news-portal
```

### Bước 2: Cấu hình biến môi trường (.env)
Sao chép file cấu hình mẫu và sử dụng các mật khẩu mạnh:
```bash
cp .env.example .env
```

### Bước 3: Khởi tạo chứng chỉ SSL tự ký (HTTPS) cho Nginx
```bash
mkdir -p nginx/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout nginx/certs/selfsigned.key   -out nginx/certs/selfsigned.crt   -subj "/C=VN/ST=ThaiNguyen/L=ThaiNguyen/O=ICTU/OU=CNTT/CN=192.168.182.129"
```

### Bước 4: Khởi chạy toàn bộ dịch vụ bằng Docker Compose
```bash
docker-compose up -d
docker-compose ps
```

---

## 4. Danh sách cổng dịch vụ (Endpoints) & Thông tin truy cập

| Dịch vụ | Container Name | Địa chỉ truy cập (URL) | Thông tin đăng nhập |
| :--- | :--- | :--- | :--- |
| **Website Tin tức (Nginx HTTPS)** | `news_nginx` / `news_wordpress` | `https://192.168.182.129` | Trang độc giả |
| **Trang Quản trị đăng bài (WP-Admin)** | `news_wordpress` | `https://192.168.182.129/wp-admin` | Tài khoản Admin WordPress |
| **Quản trị CSDL (phpMyAdmin)** | `news_phpmyadmin` | `http://192.168.182.129:8081` | User: `wp_news_user` (xem file `.env`) |
| **Prometheus Monitoring** | `news_prometheus` | `http://192.168.182.129:9090/targets` | Không yêu cầu |
| **Grafana Dashboard & LogQL** | `news_grafana` | `http://192.168.182.129:3000` | `admin` / `Grafana@Admin2026!` |
| **Loki Log Server API** | `news_loki` | `http://192.168.182.129:3100` | Kết nối nội bộ từ Grafana |

---

## 5. Giám sát (Prometheus + Grafana) & Truy vấn Log (LogQL)

### 5.1. Các Grafana Dashboard sử dụng (Data source: `http://prometheus:9090`)
- **Giám sát Docker Containers (cAdvisor):** Import Dashboard ID `21743` (hoặc `193`)
- **Giám sát Web Server (Nginx Exporter):** Import Dashboard ID `12708`
- **Giám sát Database (MySQLd Exporter):** Import Dashboard ID `7362`

### 5.2. Các câu truy vấn LogQL mẫu trên Grafana Explore (Data source: `http://loki:3100`)
- **Query 1 - Xem toàn bộ log truy cập qua Nginx Reverse Proxy:**
  ```logql
  {container="news_nginx"}
  ```
- **Query 2 - Lọc các hành vi truy cập trang quản trị đăng bài của WordPress:**
  ```logql
  {container="news_wordpress"} |= "wp-admin"
  ```
- **Query 3 - Thống kê tốc độ sinh log theo từng container trong 5 phút:**
  ```logql
  sum by (container) (rate({container=~"news_.*"}[5m]))
  ```

---

## 6. Các biện pháp Bảo mật (Hardening) đã áp dụng
1. **Non-root Containers & Chặn leo thang đặc quyền:** Các dịch vụ (`prometheus`, `grafana`, `loki`, `nginx-exporter`, `mysql-exporter`) chạy với UID non-root (`65534`, `472`, `10001`) kết hợp cờ bảo mật `security_opt: ["no-new-privileges:true"]`.
2. **Network Isolation (Cô lập mạng):** Tách biệt thành 3 mạng bridge `frontend_net`, `backend_net`, `monitoring_net`. Nginx chỉ tiếp xúc ở vùng mạng frontend, hoàn toàn không thể kết nối trực tiếp vào Database MySQL ở backend.
3. **Bảo mật mật khẩu mạnh (.env):** Sử dụng mật khẩu phức tạp trong file `.env` và chặn đẩy lên GitHub bằng file `.gitignore`.
4. **Hạn chế quyền Database (Least Privilege):** Tách biệt user ứng dụng (`wp_news_user`) và user giám sát (`exporter` chỉ được cấp quyền đọc chỉ số `SELECT, PROCESS, REPLICATION CLIENT`, giới hạn tối đa 3 kết nối đồng thời `MAX_USER_CONNECTIONS 3`).
5. **Nginx HTTPS & Security Headers:** Sử dụng chứng chỉ SSL/TLS tự ký, ẩn phiên bản Nginx (`server_tokens off;`) và cấu hình đầy đủ các header an toàn (`X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, `X-XSS-Protection: 1; mode=block`, `Strict-Transport-Security`).

