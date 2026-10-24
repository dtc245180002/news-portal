# HỆ THỐNG WEBSITE TIN TỨC / CỔNG THÔNG TIN

**Môn học:** Triển khai và Quản trị Hệ thống Phần mềm

## 1. Giới thiệu dự án

Dự án triển khai hệ thống website tin tức sử dụng WordPress, được đóng gói và vận hành bằng Docker Compose. Hệ thống tích hợp Nginx làm Reverse Proxy, MySQL làm hệ quản trị cơ sở dữ liệu, phpMyAdmin hỗ trợ quản trị cơ sở dữ liệu, cùng bộ công cụ Prometheus, Grafana và Loki phục vụ giám sát tài nguyên và thu thập nhật ký hệ thống.

### 1.1. Thông tin sinh viên

- **Họ và tên:** Lê Hồng Phong
- **Mã số sinh viên:** DTC245180002
- **Lớp:** KTPM-K23C
- **Đề tài:** Đề 19 – Website Tin tức / Cổng thông tin.

### 1.2. Mục tiêu

- Triển khai website WordPress và cơ sở dữ liệu MySQL bằng Docker Compose.
- Sử dụng Nginx làm Reverse Proxy và hỗ trợ truy cập qua HTTPS.
- Phân tách các dịch vụ bằng các mạng Docker riêng biệt.
- Quản trị cơ sở dữ liệu thông qua phpMyAdmin.
- Thu thập và giám sát metrics bằng Prometheus, Grafana, cAdvisor và các exporter.
- Thu thập nhật ký của hệ thống bằng Promtail, lưu trữ tại Loki và truy vấn thông qua Grafana.
- Áp dụng các biện pháp bảo mật phù hợp như quản lý biến môi trường, giới hạn cổng công khai và hạn chế đặc quyền của container.

## 2. Kiến trúc hệ thống

Hệ thống gồm 11 dịch vụ được quản lý bằng Docker Compose và sử dụng 3 mạng Docker bridge: `frontend_net`, `backend_net` và `monitoring_net`.

### 2.1. Chức năng các dịch vụ

| Dịch vụ | Chức năng |
|---|---|
| Nginx | Reverse Proxy, tiếp nhận yêu cầu HTTP/HTTPS và chuyển tiếp đến WordPress. |
| WordPress | Cung cấp website tin tức và trang quản trị nội dung. |
| MySQL | Lưu trữ dữ liệu website WordPress. |
| phpMyAdmin | Giao diện quản trị cơ sở dữ liệu MySQL. |
| Prometheus | Thu thập và lưu trữ metrics từ các nguồn giám sát. |
| Grafana | Hiển thị dashboard, truy vấn metrics và nhật ký hệ thống. |
| cAdvisor | Thu thập metrics về tài nguyên và hoạt động của container. |
| Nginx Exporter | Cung cấp metrics của Nginx cho Prometheus. |
| MySQL Exporter | Cung cấp metrics của MySQL cho Prometheus. |
| Loki | Tiếp nhận và lưu trữ log tập trung. |
| Promtail | Thu thập log và gửi đến Loki. |

### 2.2. Luồng hoạt động

**Luồng truy cập website:**

1. Người dùng truy cập website qua HTTP hoặc HTTPS.
2. Nginx tiếp nhận yêu cầu và chuyển tiếp đến WordPress.
3. WordPress sử dụng MySQL để đọc và ghi dữ liệu website.
4. phpMyAdmin kết nối MySQL để hỗ trợ quản trị cơ sở dữ liệu.

**Luồng giám sát metrics:**

1. cAdvisor thu thập metrics của container.
2. Nginx Exporter thu thập metrics của Nginx.
3. MySQL Exporter thu thập metrics của MySQL.
4. Prometheus định kỳ lấy metrics từ các nguồn trên.
5. Grafana truy vấn Prometheus để hiển thị dashboard.

**Luồng thu thập log:**

1. Promtail thu thập log từ các nguồn được cấu hình, bao gồm log Nginx và log container.
2. Promtail chuyển log đến Loki.
3. Grafana kết nối Loki để tìm kiếm và phân tích log bằng LogQL.

### 2.3. Phân tách mạng Docker

| Mạng | Các dịch vụ tham gia | Mục đích |
|---|---|---|
| `frontend_net` | Nginx, WordPress, Nginx Exporter | Kết nối lớp Reverse Proxy, ứng dụng web và exporter Nginx. |
| `backend_net` | WordPress, MySQL, phpMyAdmin, MySQL Exporter | Kết nối ứng dụng với cơ sở dữ liệu và các dịch vụ quản trị, giám sát MySQL. |
| `monitoring_net` | cAdvisor, Nginx Exporter, MySQL Exporter, Prometheus, Grafana, Loki, Promtail | Kết nối các thành phần giám sát và thu thập log. |

WordPress tham gia cả `frontend_net` và `backend_net` để phục vụ yêu cầu từ Nginx và kết nối MySQL. Nginx không tham gia `backend_net`, do đó không có kết nối mạng trực tiếp từ Nginx đến MySQL.

### 2.4. Cấu trúc thư mục

```text
news-portal/
├── docker-compose.yml
├── .env.example
├── .gitignore
├── README.md
├── mysql-init/
│   └── init.sql
├── nginx/
│   ├── nginx.conf
│   └── certs/
│       ├── selfsigned.crt
│       └── selfsigned.key
├── prometheus/
│   ├── prometheus.yml
│   └── .my.cnf
├── loki/
│   └── loki-config.yml
└── promtail/
    └── promtail-config.yml
```

**Chức năng các tệp và thư mục:**

- `docker-compose.yml`: Khai báo các dịch vụ, image, biến môi trường, volume, mạng và cấu hình cổng.
- `.env.example`: Mẫu các biến môi trường cần thiết để cấu hình hệ thống.
- `.env`: Chứa các giá trị cấu hình và thông tin đăng nhập thực tế; không đưa lên GitHub.
- `mysql-init/init.sql`: Khởi tạo tài khoản giám sát MySQL và cấp các quyền cần thiết khi MySQL khởi tạo cơ sở dữ liệu lần đầu.
- `nginx/nginx.conf`: Cấu hình Reverse Proxy, HTTPS và các HTTP security headers.
- `nginx/certs/`: Chứa chứng chỉ TLS và khóa riêng được Nginx sử dụng.
- `prometheus/prometheus.yml`: Khai báo các mục tiêu thu thập metrics.
- `prometheus/.my.cnf`: Chứa thông tin đăng nhập của MySQL Exporter.
- `loki/loki-config.yml`: Cấu hình Loki.
- `promtail/promtail-config.yml`: Cấu hình nguồn log và địa chỉ gửi log đến Loki.

Lưu ý: Tên và số lượng tệp thực tế có thể thay đổi nếu cấu trúc repository được cập nhật.

## 3. Yêu cầu trước khi triển khai

Môi trường triển khai cần có:

- Ubuntu Server hoặc máy ảo Ubuntu.
- Docker Engine.
- Docker Compose plugin.
- Git.
- OpenSSL để tạo chứng chỉ tự ký.
- SSH Server nếu cần truy cập các dịch vụ nội bộ từ máy Windows bằng SSH tunnel.

Kiểm tra Docker:

```bash
docker --version
docker compose version
```

Nếu chưa cài Docker hoặc Docker Compose, hãy cài đặt theo hướng dẫn chính thức của Docker dành cho hệ điều hành đang sử dụng.

## 4. Hướng dẫn cài đặt và khởi chạy

### Bước 1. Clone repository

```bash
git clone https://github.com/dtc245180002/news-portal.git
cd news-portal
```

Kiểm tra các tệp:

```bash
ls -la
```

### Bước 2. Cấu hình biến môi trường

Tạo tệp `.env` từ mẫu:

```bash
cp .env.example .env
nano .env
```

Điền các giá trị phù hợp với cấu hình trong `docker-compose.yml`. Các biến thường bao gồm:

```dotenv
MYSQL_ROOT_PASSWORD=THAY_BANG_MAT_KHAU_ROOT_MẠNH
MYSQL_DATABASE=THAY_BANG_TEN_DATABASE
MYSQL_USER=THAY_BANG_TEN_USER
MYSQL_PASSWORD=THAY_BANG_MAT_KHAU_DATABASE
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=THAY_BANG_MAT_KHAU_GRAFANA_MẠNH
```

Đây là các giá trị minh họa, không phải mật khẩu sử dụng thực tế. Nếu `.env.example` trong repository có tên biến khác hoặc bổ sung biến khác, cần giữ đúng các tên biến mà `docker-compose.yml` đang tham chiếu.

Không đưa `.env` hoặc mật khẩu thật lên GitHub.

### Bước 3. Cấu hình MySQL Exporter

Kiểm tra tệp `mysql-init/init.sql` và `prometheus/.my.cnf`.

Tệp `prometheus/.my.cnf` cần chứa thông tin đăng nhập của tài khoản `exporter` tương ứng với tài khoản được khai báo trong `mysql-init/init.sql`. Ví dụ về cấu trúc:

```ini
[client]
user=exporter
password=THAY_BANG_MAT_KHAU_EXPORTER
host=db
port=3306
```

Đây là cấu trúc minh họa; cần đảm bảo các trường phù hợp với cấu hình exporter đang sử dụng. Nếu tệp đã tồn tại, hãy cập nhật thông tin tương ứng thay vì tạo một cấu hình không khớp.

Kiểm tra quyền truy cập:

```bash
chmod 600 prometheus/.my.cnf
```

### Bước 4. Tạo chứng chỉ HTTPS tự ký

Tạo thư mục lưu chứng chỉ:

```bash
mkdir -p nginx/certs
```

Tạo chứng chỉ và khóa riêng:

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout nginx/certs/selfsigned.key \
  -out nginx/certs/selfsigned.crt \
  -subj "/C=VN/ST=ThaiNguyen/L=ThaiNguyen/O=ICTU/OU=CNTT/CN=192.168.182.129"
```

Trong lệnh trên, `192.168.182.129` là địa chỉ IP ví dụ của máy Ubuntu trong môi trường triển khai. Nếu IP của máy m khác, hãy thay bằng IP thực tế.

Phân quyền khóa riêng:

```bash
chmod 600 nginx/certs/selfsigned.key
```

**Lưu ý:** Chứng chỉ tự ký có thể khiến trình duyệt hiển thị cảnh báo bảo mật. Đây là hiện tượng bình thường khi trình duyệt chưa tin cậy chứng chỉ; chứng chỉ tự ký không tương đương chứng chỉ được CA công khai cấp.

### Bước 5. Kiểm tra cấu hình Docker Compose

Trước khi khởi chạy, kiểm tra cấu hình:

```bash
docker compose config
```

Nếu cấu hình hợp lệ, khởi động các dịch vụ:

```bash
docker compose up -d
```

Kiểm tra trạng thái:

```bash
docker compose ps
```

Xem log tổng thể:

```bash
docker compose logs --tail=100
```

Nếu một dịch vụ chưa hoạt động bình thường, hãy xem log riêng của dịch vụ đó trước khi tiếp tục.

## 5. Địa chỉ truy cập và cổng dịch vụ

Giả sử địa chỉ IP của máy Ubuntu là `192.168.182.129`:

| Dịch vụ | Cổng trên máy Ubuntu | Địa chỉ truy cập |
|---|---|---|
| Website WordPress qua Nginx | `80`, `443` | `http://192.168.182.129`, `https://192.168.182.129` |
| Trang quản trị WordPress | `443` | `https://192.168.182.129/wp-admin` |
| phpMyAdmin | `127.0.0.1:8081` | `http://localhost:8081` qua SSH tunnel |
| Prometheus | `127.0.0.1:9090` | `http://localhost:9090` qua SSH tunnel |
| Grafana | `127.0.0.1:3000` | `http://localhost:3000` qua SSH tunnel |
| Loki | Không publish cổng ra máy Ubuntu | Truy cập nội bộ qua mạng Docker |

Các cổng `8081`, `9090` và `3000` được bind vào địa chỉ loopback `127.0.0.1` của máy Ubuntu. Vì vậy, không thể mặc định truy cập chúng trực tiếp từ máy Windows bằng địa chỉ IP của Ubuntu.

### 5.1. Truy cập website

Mở trình duyệt và truy cập:

```text
https://192.168.182.129
```

Để vào trang quản trị WordPress:

```text
https://192.168.182.129/wp-admin
```

Sử dụng tài khoản quản trị WordPress được tạo trong quá trình cài đặt website. Tài khoản này không nhất thiết trùng với tài khoản MySQL hoặc Grafana.

### 5.2. Truy cập phpMyAdmin, Prometheus và Grafana từ Windows

Mở PowerShell trên Windows và chạy:

```bash
ssh -N -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 -L 8081:127.0.0.1:8081 TEN_USER_SSH@192.168.182.129
```

Thay `TEN_USER_SSH` bằng tên tài khoản SSH trên Ubuntu.

Giữ cửa sổ PowerShell hoạt động trong khi sử dụng các dịch vụ. Sau khi SSH tunnel được thiết lập, mở trình duyệt Windows:

- Grafana: `http://localhost:3000`
- Prometheus: `http://localhost:9090/targets`
- phpMyAdmin: `http://localhost:8081`

Để ngắt kết nối, nhấn `Ctrl + C` trong cửa sổ PowerShell đang chạy SSH tunnel.

Nếu chỉ cần sử dụng Grafana và Prometheus, có thể bỏ phần `-L 8081:127.0.0.1:8081`.

### 5.3. Tài khoản đăng nhập

- **WordPress:** Tài khoản quản trị được tạo khi thiết lập WordPress.
- **phpMyAdmin:** Sử dụng tài khoản MySQL được cấu hình trong `.env`.
- **Grafana:** Sử dụng `GRAFANA_ADMIN_USER` và `GRAFANA_ADMIN_PASSWORD` trong `.env`. Tài khoản mặc định được cấu hình có thể là `admin`, tùy giá trị thực tế.
- **Prometheus:** Trong cấu hình hiện tại, cổng được giới hạn ở loopback và không yêu cầu đăng nhập riêng theo cấu hình mặc định.

Không lưu mật khẩu thật trong README.

## 6. Cấu hình giám sát bằng Prometheus và Grafana

### 6.1. Kiểm tra Prometheus

Mở:

```text
http://localhost:9090/targets
```

Địa chỉ này giả định SSH tunnel đã được thiết lập.

Trang Targets hiển thị trạng thái thu thập metrics từ các mục tiêu đã khai báo trong `prometheus/prometheus.yml`.

Kiểm tra các mục tiêu được cấu hình và xác nhận chúng có trạng thái `UP`. Nếu mục tiêu ở trạng thái `DOWN`, kiểm tra địa chỉ scrape, cấu hình exporter, kết nối mạng Docker và log của dịch vụ liên quan.

### 6.2. Cấu hình Data Sources trong Grafana

Đăng nhập Grafana, sau đó vào phần cấu hình Data Sources.

**Prometheus**

- Type: Prometheus
- URL: `http://prometheus:9090`

**Loki**

- Type: Loki
- URL: `http://loki:3100`

Các URL trên sử dụng tên dịch vụ Docker và cổng nội bộ. Chúng chỉ hoạt động khi các dịch vụ tương ứng có thể kết nối với nhau qua mạng Docker và cấu hình Data Source chính xác.

Sau khi cấu hình, sử dụng chức năng kiểm tra kết nối của Grafana để xác nhận Data Source hoạt động.

### 6.3. Dashboard giám sát

Có thể tham khảo các dashboard cộng đồng sau nếu chúng tương thích với phiên bản và loại metrics của hệ thống:

| Thành phần | Dashboard tham khảo |
|---|---|
| Docker và container | ID `21743` hoặc `193` |
| Nginx | ID `12708` |
| MySQL | ID `7362` |

Các ID trên là dashboard tham khảo, không đảm bảo tự động tương thích với mọi phiên bản exporter hoặc cấu hình metrics. Nếu dashboard không hiển thị dữ liệu, hãy kiểm tra Data Source, tên metrics và phiên bản dashboard.

## 7. Thu thập và truy vấn log bằng Loki

Promtail được cấu hình để thu thập log từ các nguồn đã khai báo và gửi về Loki. Grafana sử dụng Data Source Loki để tìm kiếm log bằng LogQL.

Vào Grafana → Explore, chọn Data Source Loki và thử các truy vấn sau.

### 7.1. Xem log truy cập Nginx

```logql
{container="news_nginx"}
```

Truy vấn này chỉ trả về kết quả nếu log được thu thập và có nhãn `container="news_nginx"` như cấu hình Promtail thực tế.

### 7.2. Tìm log có chứa `wp-admin`

```logql
{container="news_wordpress"} |= "wp-admin"
```

Truy vấn tìm các dòng log thuộc stream có nhãn `container="news_wordpress"` và chứa chuỗi `wp-admin`. Tuy nhiên, log truy cập HTTP thường được ghi ở Nginx; nếu WordPress không ghi loại log này, hãy tìm trong stream Nginx thay vì WordPress.

### 7.3. Thống kê tốc độ sinh log

```logql
sum by (container) (rate({container=~"news_.*"}[5m]))
```

Truy vấn tính tốc độ log theo nhãn `container` trong khoảng thời gian 5 phút. Kết quả phụ thuộc vào các nhãn thực tế mà Promtail gán cho log.

Nếu không có dữ liệu, hãy kiểm tra cấu hình Promtail, trạng thái Loki, thời gian truy vấn trong Grafana và các nhãn log hiện có.

## 8. Các biện pháp bảo mật đã áp dụng

### 8.1. Giới hạn cổng truy cập

- Nginx publish cổng `80` và `443` để phục vụ website.
- phpMyAdmin bind vào `127.0.0.1:8081`.
- Prometheus bind vào `127.0.0.1:9090`.
- Grafana bind vào `127.0.0.1:3000`.
- Loki không publish cổng trực tiếp ra máy Ubuntu.

Các dịch vụ bind vào loopback chỉ được truy cập trực tiếp từ máy Ubuntu hoặc qua cơ chế chuyển tiếp phù hợp như SSH tunnel.

### 8.2. Phân tách mạng

Hệ thống sử dụng ba mạng Docker: `frontend_net`, `backend_net` và `monitoring_net`. Việc phân tách giúp giới hạn kết nối giữa các nhóm dịch vụ theo cấu hình mạng.

WordPress tham gia cả mạng frontend và backend; Nginx chỉ tham gia mạng frontend. Vì vậy, Nginx không có kết nối mạng Docker trực tiếp đến MySQL.

### 8.3. Quản lý mật khẩu và thông tin nhạy cảm

- Sử dụng `.env` để cung cấp biến môi trường.
- Không commit `.env` và `prometheus/.my.cnf` chứa thông tin xác thực thật.
- Duy trì `.env.example` với giá trị minh họa, không chứa bí mật đang sử dụng.
- Hạn chế quyền đọc đối với tệp chứa mật khẩu khi phù hợp.

### 8.4. Hạn chế đặc quyền container

Một số dịch vụ giám sát được cấu hình chạy với UID không phải root, kết hợp `no-new-privileges:true`.

Cấu hình này giúp giảm một số rủi ro liên quan đến đặc quyền, nhưng không đồng nghĩa mọi container đều chạy non-root. Cần đánh giá từng dịch vụ theo cấu hình thực tế.

### 8.5. Quyền tài khoản MySQL Exporter

Tài khoản `exporter` được tách khỏi tài khoản ứng dụng WordPress và được cấp các quyền giám sát cần thiết, thay vì sử dụng tài khoản root.

Cấu hình mẫu có thể sử dụng các quyền `PROCESS`, `REPLICATION CLIENT`, `SELECT` và giới hạn kết nối đồng thời. Cần xác nhận `mysql-init/init.sql` và cấu hình xác thực thực tế khớp nhau.

Lưu ý: Script khởi tạo của image MySQL thường chỉ chạy khi khởi tạo thư mục dữ liệu lần đầu. Nếu volume MySQL đã tồn tại, sửa `init.sql` không tự động cập nhật quyền hoặc mật khẩu của tài khoản đã được tạo trước đó.

### 8.6. HTTPS và HTTP Security Headers

Nginx được cấu hình HTTPS với chứng chỉ tự ký trong môi trường thử nghiệm. Cấu hình có thể bao gồm việc ẩn phiên bản Nginx và thiết lập các header như:

- `X-Frame-Options`
- `X-Content-Type-Options`
- `X-XSS-Protection`
- `Strict-Transport-Security`

Hiệu quả thực tế phụ thuộc vào nội dung `nginx/nginx.conf`. Chỉ nên mô tả các header là đã được triển khai sau khi kiểm tra cấu hình và phản hồi HTTP tương ứng. Khi triển khai thực tế, cần xem xét chứng chỉ hợp lệ, phạm vi áp dụng HSTS và các yêu cầu bảo mật khác.

## 9. Vận hành và xử lý sự cố

Các lệnh dưới đây được chạy trong thư mục chứa `docker-compose.yml`.

### 9.1. Kiểm tra trạng thái dịch vụ

```bash
docker compose ps
```

### 9.2. Xem log của một dịch vụ

Ví dụ:

```bash
docker compose logs --tail=100 nginx
docker compose logs --tail=100 wordpress
docker compose logs --tail=100 db
docker compose logs --tail=100 prometheus
docker compose logs --tail=100 grafana
docker compose logs --tail=100 loki
docker compose logs --tail=100 promtail
```

Có thể thay tên dịch vụ để xem log của các thành phần khác.

### 9.3. Theo dõi log trực tiếp

```bash
docker compose logs -f
```

Nhấn `Ctrl + C` để ngừng theo dõi log. Thao tác này không có nghĩa là dừng các container.

### 9.4. Khởi động lại dịch vụ

```bash
docker compose restart
```

Hoặc khởi động lại một dịch vụ cụ thể:

```bash
docker compose restart nginx
```

### 9.5. Dừng hệ thống

```bash
docker compose down
```

Lệnh này dừng và xóa các container cùng mạng do Compose quản lý, nhưng thông thường không xóa các named volume đã khai báo. Dữ liệu WordPress và MySQL có thể tiếp tục được lưu trong volume.

**Không chạy `docker compose down -v` nếu chưa hiểu rõ hậu quả**, vì tùy cấu hình, lệnh này có thể xóa các named volume và làm mất dữ liệu lưu trong chúng.

### 9.6. Một số lỗi thường gặp

| Hiện tượng | Hướng kiểm tra |
|---|---|
| Website không truy cập được | Kiểm tra `docker compose ps`, log Nginx/WordPress, IP máy Ubuntu và cổng 80/443. |
| WordPress không kết nối được MySQL | Kiểm tra `WORDPRESS_DB_HOST`, tên database, tài khoản và mật khẩu; kiểm tra log `db` và `wordpress`. |
| Prometheus có target `DOWN` | Kiểm tra `prometheus.yml`, tên dịch vụ, cổng nội bộ và khả năng kết nối mạng. |
| Grafana không có dữ liệu | Kiểm tra Data Source, kết nối Prometheus/Loki, khoảng thời gian và truy vấn. |
| Loki không có log | Kiểm tra log Promtail/Loki, đường dẫn log, cấu hình nhãn và quyền đọc các nguồn log. |
| Không mở được Grafana từ Windows | Kiểm tra SSH tunnel, tài khoản SSH và cổng `127.0.0.1:3000` trên Ubuntu. |
| Trình duyệt cảnh báo chứng chỉ | Kiểm tra chứng chỉ tự ký và địa chỉ IP/tên máy được sử dụng khi truy cập. |

## 10. Kiểm tra sau triển khai

Có thể sử dụng danh sách dưới đây để nghiệm thu hệ thống:

- [ ] Các dịch vụ khởi động theo cấu hình Docker Compose.
- [ ] Website WordPress truy cập được qua Nginx.
- [ ] Trang quản trị WordPress hoạt động.
- [ ] WordPress kết nối được MySQL.
- [ ] phpMyAdmin truy cập được qua SSH tunnel và đăng nhập được.
- [ ] Prometheus truy cập được và các target cần thiết có trạng thái `UP`.
- [ ] Grafana kết nối được với Prometheus.
- [ ] Grafana kết nối được với Loki.
- [ ] Promtail gửi được log đến Loki.
- [ ] Có thể truy vấn log bằng LogQL.
- [ ] Các cổng quản trị được giới hạn theo cấu hình mong muốn.
- [ ] Các tệp chứa mật khẩu và khóa riêng không bị commit lên repository.

## 11. Tài liệu tham khảo

- Docker Docs: https://docs.docker.com/
- Docker Compose: https://docs.docker.com/compose/
- WordPress: https://wordpress.org/documentation/
- Nginx: https://nginx.org/en/docs/
- Prometheus: https://prometheus.io/docs/
- Grafana: https://grafana.com/docs/
- Loki: https://grafana.com/docs/loki/
- Promtail: https://grafana.com/docs/loki/latest/send-data/promtail/
- MySQL: https://dev.mysql.com/doc/

---

**Ghi chú:** Các địa chỉ IP, tên tài khoản, đường dẫn, tên nhãn log và các thông số cấu hình trong tài liệu cần được đối chiếu với môi trường triển khai thực tế trước khi sử dụng làm tài liệu nghiệm thu hoặc báo cáo cuối cùng.
