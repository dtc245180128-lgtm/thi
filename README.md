# DỰ ÁN THỰC HÀNH TỔNG HỢP: XÂY DỰNG, BẢO MẬT VÀ GIÁM SÁT HỆ THỐNG MẠNG MICROSERVICES DỰ TRÊN DOCKER

---

## 👨‍🎓 THÔNG TIN SINH VIÊN VÀ ĐỀ TÀI

* **Trường:** Đại học CNTT & Truyền thông - Đại học Thái Nguyên (ICTU)
* **Khoa:** Công nghệ Thông tin
* **Sinh viên thực hiện:** Cao Tiến Đạt
* **Mã số sinh viên:** DTC245180128
* **Lớp:** K24
* **Tên Repository GitHub:** `bds-dtc245180128`
* **Hệ điều hành thử nghiệm:** Ubuntu Server 22.04 LTS (Chạy trên máy ảo VMware/Hyper-V)

---

## 📐 1. KIẾN TRÚC HỆ THỐNG (SYSTEM ARCHITECTURE)

Hệ thống được thiết kế hoàn toàn theo mô hình **Microservices** hóa, sử dụng **Docker Compose** để đóng gói, quản lý và vận hành 11 dịch vụ riêng biệt. Hệ thống được chia làm 3 vùng mạng độc lập (Network Isolation) để đảm bảo an toàn thông tin.

```text
                                  +---------------------------------------+
                                  |            CLIENT / BROWSER           |
                                  +---------------------------------------+
                                                      |
                                         [HTTP Port 80 / 8085 / 3000]
                                                      |
                                                      v
  ===================================== FRONTEND NETWORK (frontend_net) =====================================
                                                      |
                            +-------------------------+-------------------------+
                            |                                                   |
                            v                                                   v
                +-----------------------+                           +-----------------------+
                |   nginx_proxy (80)    |                           |  wordpress_app (8085) |
                +-----------------------+                           +-----------------------+
                            |                                                   |
  ====================================== BACKEND NETWORK (backend_net) =====================================
                            |                                                   |
            +---------------+---------------+                                   |
            |               |               |                                   |
            v               v               v                                   v
    +---------------+ +-----------+ +---------------+                   +---------------+
    |    web_app    | |db_adminer | |    grafana    |                   | wordpress_db  |
    | (VietHomes)   | | (Adminer) | |  (Dashboard)  |                   |    (MySQL)    |
    +---------------+ +-----------+ +---------------+                   +---------------+
            |                               |                                   
            v                               |                                   
    +---------------+                       |                                   
    |  postgres_db  |                       |                                   
    | (PostgreSQL)  |                       |                                   
    +---------------+                       |                                   
                                            |                                   
  ==================================== MONITORING NETWORK (monitoring_net) =================================
                                            |                                   
            +-------------------------------+-------------------------------+
            |                               |                               |
            v                               v                               v
    +---------------+               +---------------+               +---------------+
    |  prometheus   |               |   cadvisor    |               | loki (3100) / |
    | (Metrics 9090)|               |  (Resources)  |               |   promtail    |
    +---------------+               +---------------+               +---------------+
```

---

## 🛠️ 2. BẢNG CHI TIẾT 11 DỊCH VỤ CONTAINER (SERVICES MATRIX)

| STT | Tên Container | Hình ảnh (Image) | Cổng Out:In | Mạng Tham Gia | Vai Trò & Chức Năng Chi Tiết |
| :---: | :--- | :--- | :---: | :--- | :--- |
| **1** | `nginx_proxy` | `nginx:alpine` | `80:80` | `frontend_net`, `backend_net` | **Reverse Proxy:** Điểm tiếp nhận lưu lượng duy nhất từ ngoài vào Web app, chèn Security Headers. |
| **2** | `web_app` | `custom build` | Nội bộ | `backend_net` | **Web Bất Động Sản VietHomes:** Ứng dụng chính hiển thị danh sách dự án bất động sản. |
| **3** | `postgres_db` | `postgres:15-alpine` | Nội bộ | `backend_net` | **Database:** Lưu trữ dữ liệu dự án, khách hàng cho ứng dụng VietHomes. |
| **4** | `db_adminer` | `adminer:latest` | Nội bộ | `backend_net` | **DB Management Tool:** Giao diện Web quản trị CSDL PostgreSQL/MySQL. |
| **5** | `prometheus` | `prom/prometheus:v2.45` | `9090:9090` | `monitoring_net` | **Metrics Collector:** Thu thập chỉ số hiệu năng (CPU, RAM, Network) từ cAdvisor. |
| **6** | `grafana` | `grafana/grafana:latest` | `3000:3000` | `monitoring_net`, `backend_net` | **Visualizer:** Trực quan hóa dữ liệu chỉ số và hiển thị nhật ký Log tập trung. |
| **7** | `cadvisor` | `gcr.io/cadvisor/cadvisor` | Nội bộ | `monitoring_net` | **Container Monitor:** Trích xuất thông số tiêu thụ tài nguyên phần cứng của từng Container. |
| **8** | `loki` | `grafana/loki:2.8.2` | `3100:3100` | `monitoring_net` | **Log Aggregator:** Hệ thống lưu trữ và index nhật ký (Log) tập trung. |
| **9** | `promtail` | `grafana/promtail:2.8.2` | Nội bộ | `monitoring_net` | **Log Collector:** Đọc file log từ Docker `/var/lib/docker/containers` và đẩy về Loki. |
| **10** | `wordpress_db` | `mysql:5.7` | Nội bộ | `backend_net` | **Database MySQL:** Cơ sở dữ liệu dành riêng cho dịch vụ WordPress. |
| **11** | `wordpress_app` | `wordpress:latest` | `8085:80` | `frontend_net`, `backend_net` | **WordPress CMS:** Trang quản trị tài khoản, người dùng và xuất bản bài viết tin tức. |

---

## 🔒 3. CÁC BIỆN PHÁP BẢO MẬT & TỐI ƯU (SECURITY HARDENING)

### 3.1. Cô Lập Mạng Nội Bộ (Network Isolation)
* Hệ thống khởi tạo 3 phân vùng mạng Docker Bridge độc lập: `frontend_net`, `backend_net` và `monitoring_net`.
* Mạng `backend_net` được thêm thuộc tính `internal: true`. Điều này **ngăn chặn hoàn toàn các Database (`postgres_db`, `wordpress_db`) tiếp xúc trực tiếp với môi trường Internet bên ngoài**, giảm thiểu nguy cơ bị tấn công Brute-force hoặc Exploit cổng DB.

### 3.2. Giới Hạn Quyền Người Dùng (Non-root Container)
* Các ứng dụng Web (Nginx Proxy, Web App) được cấu hình chạy dưới tài khoản giới hạn quyền (`nginx` hoặc `www-data`), không chạy trực tiếp bằng quyền `root` của Linux Host. Khi container bị chiếm quyền điều khiển, sự cố không thể leo thang sang máy chủ thật.

### 3.3. Tích Hợp Các Header Bảo Mật (Nginx Security Headers)
Nginx Reverse Proxy được bổ sung các chỉ thị bảo mật trực tiếp trong file `nginx.conf`:
* `X-Frame-Options "SAMEORIGIN"`: Chống tấn công chèn khung ẩn (Clickjacking).
* `X-XSS-Protection "1; mode=block"`: Bật bộ lọc chống tấn công kịch bản chéo trang (Cross-Site Scripting - XSS).
* `X-Content-Type-Options "nosniff"`: Ngăn chặn trình duyệt đoán sai kiểu MIME (MIME-sniffing).
* `Referrer-Policy "no-referrer-when-downgrade"`: Bảo vệ thông tin liên kết chuyển hướng.

### 3.4. Quản Lý Mật Khẩu Mạnh (Strong Credentials)
* Toàn bộ tài khoản CSDL (`PostgreSQL`, `MySQL`) được gán mật khẩu phức tạp chứa tối thiểu 12 ký tự gồm chữ hoa, chữ thường, chữ số và ký tự đặc biệt (Ví dụ: `StrongPassword123!@#`).

---

## 🚀 4. HƯỚNG DẪN KHỞI CHẠY HỆ THỐNG CHI TIẾT

### Yêu Cầu Tiên Quyết
* Máy chủ Linux Ubuntu 20.04/22.04 LTS có kết nối Internet.
* Đã cài đặt `docker` (v20.10+) và `docker-compose` (v2.0+).

### Các Bước Triển Khai

1. **Clone dự án từ GitHub về máy chủ:**
   ```bash
   cd ~
   git clone https://github.com/caodat/bds-dtc245180128.git
   cd bds-dtc245180128
   ```

2. **Cấp quyền truy cập Log Docker cho Promtail:**
   ```bash
   sudo chmod -R 755 /var/lib/docker/containers/
   ```

3. **Khởi chạy cụm Microservices ở chế độ chạy ngầm (Detached mode):**
   ```bash
   docker-compose up -d
   ```

4. **Xác nhận trạng thái hoạt động của 11 Containers:**
   ```bash
   docker-compose ps
   ```
   *(Yêu cầu tất cả 11 dịch vụ đều báo trạng thái `Up` hoặc `running`)*.

---

## 🌐 5. DANH SÁCH CỔNG & ĐƯỜNG DẪN TRUY CẬP

*(Thay `192.168.152.130` bằng địa chỉ IP Server thực tế của bạn)*

* **Website Bất Động Sản VietHomes:** `http://192.168.152.130/`
* **Trang Quản Trị WordPress CMS:** `http://192.168.152.130:8085/`
* **Bảng Điều Khiển Giám Sát Grafana:** `http://192.168.152.130:3000/` *(Mặc định: `admin` / `admin`)*
* **Hệ Thống Thu Thập Metrics Prometheus:** `http://192.168.152.130:9090/`
* **Hệ Thống Log Tập Trung Loki API:** `http://192.168.152.130:3100/`

---

## 📊 6. HƯỚNG DẪN GIÁM SÁT & TRUY VẤN LOGQL

### 6.1. Giám Sát Container Bằng Grafana Dashboard
1. Truy cập Grafana tại `http://192.168.152.130:3000`.
2. Vào mục **Dashboards** $\rightarrow$ **New** $\rightarrow$ **Import**.
3. Nhập ID Dashboard công khai: **`14282`** (hoặc **`893`**) $\rightarrow$ Nhấn **Load**.
4. Chọn Data source đầu vào là **Prometheus** $\rightarrow$ Nhấn **Import**.

### 6.2. Truy Vấn Nhật Ký Log Bằng Ngôn Ngữ LogQL
Truy cập **Grafana** $\rightarrow$ **Explore** $\rightarrow$ Chọn Data Source **Loki**:

* **Truy vấn 1: Trích xuất toàn bộ Log từ các Container đang chạy:**
  ```logql
  {job="container_logs"}
  ```

* **Truy vấn 2: Lọc các dòng Log liên quan tới yêu cầu truy cập HTTP GET:**
  ```logql
  {job="container_logs"} |= "GET"
  ```

* **Truy vấn 3: Lọc các dòng Log cảnh báo lỗi HTTP Client (4xx) hoặc Server (5xx):**
  ```logql
  {job="container_logs"} |~ "4[0-9]{2}|5[0-9]{2}"
  ```

---

## 📋 7. QUY TRÌNH QUẢN LÝ MÃ NGUỒN & LỊCH SỬ COMMIT

Dự án được phân chia và quản lý theo 3 mốc commit chuẩn trên GitHub:

* **Commit 1:** `Triển khai Nginx Reverse Proxy và Web App VietHomes`  
  *(Cấu hình `docker-compose.yml`, ứng dụng Web, CSDL PostgreSQL và Nginx Proxy với Security Headers)*.
* **Commit 2:** `Tích hợp Prometheus, Grafana và cAdvisor giám sát container`  
  *(Thêm các dịch vụ Prometheus, Grafana, cAdvisor vào hệ thống giám sát tài nguyên)*.
* **Commit 3:** `Cấu hình Loki LogQL tập trung và tích hợp WordPress`  
  *(Bổ sung Promtail, Loki quản lý log tập trung, tích hợp WordPress CMS và MySQL)*.

---

## 🧪 8. BỘ LỆNH KIỂM THỬ KHI BÁO CÁO (VERIFICATION COMMANDS)

```bash
# 1. Kiểm tra Security Headers trả về từ Nginx Proxy
curl -I http://localhost

# 2. Kiểm tra người dùng thực thi trong Container (Minh chứng Non-root)
docker exec -it web_app whoami

# 3. Kiểm tra danh sách các Mạng Docker
docker network ls

# 4. Kiểm tra chi tiết Mạng backend_net (Xác nhận thuộc tính internal)
docker network inspect bds-dtc245180128_backend_net

# 5. Xem Log trực tiếp của Loki
docker logs loki
