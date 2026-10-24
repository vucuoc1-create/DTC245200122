# 🚀 HỆ THỐNG BLOG / CMS ENTERPRISE (DEVOPS & HARDENED STACK)

Dự án triển khai một hạ tầng hoàn chỉnh cho nền tảng CMS / Blog (WordPress) được container hóa bằng Docker Compose, tích hợp cổng Nginx Reverse Proxy gia cố bảo mật, hệ thống giám sát Metrics (Prometheus + Grafana), và hệ thống thu thập Log tập trung (Loki + Promtail).

---

## 📌 1. KIẾN TRÚC TỔNG THỂ HỆ THỐNG

```
                                  [ Client / Browser ]
                                           │
                                    HTTP Port 80
                                           ▼
                               ┌──────────────────────┐
                               │     blog_nginx       │  <── Security Headers, server_tokens off
                               │  (Nginx Rev. Proxy)  │  <── client_max_body_size 64M
                               └──────────┬───────────┘
                                          │ (Internal Network: blog_network)
                    ┌─────────────────────┴─────────────────────┐
                    ▼                                           ▼
         ┌────────────────────┐                      ┌────────────────────┐
         │   blog_wordpress   │                      │  blog_phpmyadmin   │ (Port 8081)
         │ (WordPress CMS)    │                      │ (Web DB Manager)   │
         └──────────┬─────────┘                      └──────────┬─────────┘
                    │                                           │
                    └─────────────────────┬─────────────────────┘
                                          │
                                          ▼
                               ┌──────────────────────┐
                               │      blog_mysql      │ (MySQL 8.0 - Port 3306 KHÔNG public)
                               │  (Database Storage)  │ <── Named Volume: db_data
                               └──────────────────────┘

 ════════════════════════════════════════════════════════════════════════════════════════
   GIÁM SÁT METRICS & LOG TẬP TRUNG (OBSERVABILITY STACK)
 ════════════════════════════════════════════════════════════════════════════════════════
   • Promtail Agent: Thu thập Docker stream logs & Nginx access/error logs ──► Đẩy về Loki (3100)
   • Prometheus: Định kỳ scrape metrics của hạ tầng ──► Lưu trữ Time-series TSDB (9090)
   • Grafana: Dashboard trực quan hóa cả Metrics & Logs (Port 3000, Auto-provisioning Datasources)
```

---

## 🔒 2. CÁC BIỆN PHÁP BẢO MẬT ĐÃ ĐƯỢC GIA CỐ (SECURITY HARDENING)

Dự án áp dụng các nguyên tắc phòng thủ theo chiều sâu (Defense-in-Depth):

1. **Cách ly mạng nội bộ (Network Isolation):**
   - Thiết lập custom Docker bridge network (`blog_network`).
   - Cổng MySQL (`3306`) và cổng Apache của WordPress (`80`) **HOÀN TOÀN KHÔNG BỊ EXPOSE RA MÁY CHỦ HOST**. Chỉ các container trong cùng mạng `blog_network` mới có quyền kết nối.
2. **Reverse Proxy & Security Headers (Nginx):**
   - Mọi truy cập vào WordPress bắt buộc phải đi qua Nginx.
   - Thêm các Security Headers chuẩn OWASP:
     - `X-Frame-Options: SAMEORIGIN` (Chống Clickjacking).
     - `X-Content-Type-Options: nosniff` (Chống MIME-type sniffing).
     - `X-XSS-Protection: 1; mode=block` (Kích hoạt bộ lọc XSS).
     - `Referrer-Policy: strict-origin-when-cross-origin` (Bảo vệ thông tin chuyển hướng).
     - `Content-Security-Policy: default-src 'self' ...` (Kiểm soát nguồn tài nguyên tải về).
   - Ẩn định danh và phiên bản web server: `server_tokens off;` (Chống OSINT/Banner Grabbing).
   - Giới hạn payload upload an toàn: `client_max_body_size 64M;`.
3. **Quản lý bí mật & Cấu hình môi trường (.env):**
   - Toàn bộ mật khẩu nhạy cảm (`MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`, `GRAFANA_ADMIN_PASSWORD`) đều được sinh với độ phức tạp cao và lưu trữ trong file `.env`.
   - File `.gitignore` được cấu hình để **chặn tuyệt đối** việc commit file `.env` lên Git Repository. File `.env.example` cung cấp bản mẫu an toàn.
4. **Phòng chống cạn kiệt tài nguyên đĩa (Disk Exhaustion Mitigation):**
   - Cấu hình logging driver `json-file` trên toàn bộ container với log rotation:
     - `max-size: "10m"`
     - `max-file: "3"`
     - Giúp hệ thống không bao giờ bị đầy ổ đĩa do log container phình to.
5. **Độ tin cậy & Sẵn sàng (Healthchecks):**
   - Khởi tạo `mysqladmin ping` healthcheck trên MySQL để WordPress và phpMyAdmin chỉ khởi động sau khi Database đã hoàn toàn sẵn sàng nhận kết nối.

---

## 🌐 3. BẢNG DANH MỤC DỊCH VỤ, PORT & THÔNG TIN ĐĂNG NHẬP

| Dịch vụ | Địa chỉ truy cập (URL) | Port | Thông tin đăng nhập mặc định |
| :--- | :--- | :--- | :--- |
| **Blog WordPress** (qua Nginx) | [http://localhost](http://localhost) | `80` | Thiết lập tài khoản admin lần đầu qua giao diện web |
| **phpMyAdmin** | [http://localhost:8081](http://localhost:8081) | `8081` | Server: `db`<br>Username: `wp_user` (hoặc `root`)<br>Password: `wp123` (hoặc root: `root123`) |
| **Grafana Dashboard** | [http://localhost:3000](http://localhost:3000) | `3000` | Username: `admin`<br>Password: `admin123` |
| **Prometheus UI** | [http://localhost:9090](http://localhost:9090) | `9090` | Không cần xác thực (Internal monitoring) |
| **Loki Centralized Logs** | [http://localhost:3100/ready](http://localhost:3100/ready) | `3100` | HTTP API đẩy/truy vấn log |

> **Lưu ý đặc biệt:** Datasource Prometheus và Loki đã được **tự động cấu hình sẵn (auto-provisioned)** vào Grafana qua thư mục `grafana/provisioning/datasources/datasources.yml`. Khi đăng nhập vào Grafana, bạn không cần phải thêm datasource thủ công!

---

## ⚡ 4. HƯỚNG DẪN KHỞI CHẠY HỆ THỐNG

### Bước 1: Chuẩn bị file biến môi trường
Nếu tải dự án về một máy mới, sao chép file `.env.example` thành `.env`:
```powershell
cp .env.example .env
```
*(Nếu đã có sẵn file `.env`, bạn có thể kiểm tra hoặc tùy chỉnh các mật khẩu theo ý muốn).*

### Bước 2: Khởi động toàn bộ cụm dịch vụ
Chạy lệnh sau tại thư mục gốc của dự án:
```powershell
docker-compose up -d
```
*(Hoặc `docker compose up -d`)*

### Bước 3: Kiểm tra trạng thái hoạt động của các container
```powershell
docker-compose ps
```
Tất cả các container (`blog_nginx`, `blog_wordpress`, `blog_mysql`, `blog_phpmyadmin`, `blog_prometheus`, `blog_grafana`, `blog_loki`, `blog_promtail`) đều phải ở trạng thái `Up` hoặc `running`.

### Bước 4: Tắt hệ thống khi kết thúc
```powershell
docker-compose down
```
*(Nếu muốn xóa sạch toàn bộ volume dữ liệu để làm lại từ đầu: `docker-compose down -v`)*

---

## 🔍 5. TRUY VẤN LOG TẬP TRUNG (LOGQL DEMO TRÊN GRAFANA)

Để demo nghiệm thu hệ thống Log tập trung với Giáo viên:
1. Truy cập Grafana tại `http://localhost:3000`, đăng nhập bằng `admin` / `admin123`.
2. Chọn menu **Explore** (biểu tượng la bàn ở thanh menu bên trái).
3. Chọn Data source: **Loki**.
4. Sử dụng các câu truy vấn **LogQL** mẫu dưới đây:

### Câu lệnh 1: Xem toàn bộ Access Log & Error Log của Nginx Reverse Proxy
```logql
{job="nginx"}
```
*Tác dụng: Hiển thị thời gian thực tất cả lượt truy cập của người dùng đến blog thông qua Nginx, bao gồm mã trạng thái HTTP (200, 301, 404, 500), IP máy trỏ đến.*

### Câu lệnh 2: Lọc các request dạng HTTP GET hoặc POST gửi vào hệ thống
```logql
{job="nginx"} |= "GET"
```
*Tác dụng: Lọc trực tiếp các bản ghi log có chứa chuỗi "GET". Bạn có thể đổi thành `|= "POST"` để kiểm tra các hành động gửi form, login.*

### Câu lệnh 3: Lọc log của Database MySQL
```logql
{container="blog_mysql"}
```
*Tác dụng: Theo dõi toàn bộ quá trình khởi động, đọc cấu hình và các cảnh báo từ MySQL 8.0.*

### Câu lệnh 4: Truy tìm sự cố (Troubleshooting) toàn bộ cụm container
```logql
{job="docker"} |= "error"
```
*(Hoặc `{job=~".+"} |= "error"`)*
*Tác dụng: Quét tất cả các log streams trên toàn bộ hệ thống để phát hiện nhanh các thông báo lỗi phát sinh.*
