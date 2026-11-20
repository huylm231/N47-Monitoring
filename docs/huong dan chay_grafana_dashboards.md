# HƯỚNG DẪN MỞ & VẬN HÀNH GRAFANA DASHBOARDS (TV4)
> **Dự án:** N4.7 – Hệ thống giám sát máy chủ, Cloud & TT dữ liệu  
> **Người thực hiện:** Trương Tấn An (TV4 – Grafana & Dashboard Specialist)

---

## 1. Yêu cầu tiên quyết & Khởi động hệ thống

Chỉ cần máy tính đã cài đặt **Docker Desktop** và clone source code dự án `N47-Monitoring`.

Mở terminal tại thư mục gốc dự án và chạy:
```bash
# Khởi động cụm dịch vụ giám sát chạy ngầm
docker compose up -d

# Kiểm tra đảm bảo 4 container đều Up
docker compose ps
```
> Đảm bảo các container `grafana` (cổng 3000), `prometheus` (cổng 9090), `node-exporter` (cổng 9100) và `cadvisor` (cổng 8080) đang hoạt động.

---

## 2. Đăng nhập Grafana

1. Mở trình duyệt web truy cập: [http://localhost:3000](http://localhost:3000)
2. Nhập thông tin đăng nhập mặc định:
   * **Username:** `admin`
   * **Password:** `admin12345`

---

## 3. Cách mở các Dashboard đã dựng

### Cách 1: Xem trực tiếp các Dashboard có sẵn (Khuyên dùng)
Trên thanh menu bên trái:
1. Nhấp chọn **Dashboards**.
2. Chọn bảng điều khiển bạn muốn theo dõi:
   * **`Docker Container Overview`**: Bảng giám sát chi tiết tài nguyên từng container (CPU, RAM, Lưu lượng mạng Rx/Tx) và hỗ trợ lọc động.
   * **`Alerts & Services Status`**: Bảng theo dõi trạng thái sống còn của các dịch vụ (`ONLINE`/`OFFLINE`) và số lượng cảnh báo bùng phát (`Firing Alerts`).
   * **`System Overview Dashboard`**: Bảng giám sát phần cứng máy chủ host (CPU, RAM, Disk, Network, Uptime).

---

### Cách 2: Nhập thủ công từ file mã nguồn (Khi chạy trên máy mới hoàn toàn)
Nếu chạy trên một môi trường Grafana mới tinh chưa có dữ liệu sẵn:
1. Vào **Dashboards** > Nhấn nút **New** (góc trên bên phải) > Chọn **Import**.
2. Nhấp nút **Upload dashboard JSON file**.
3. Chọn file mã nguồn được lưu tại:
   ```text
   grafana/dashboards/dashboard.json
   ```
4. Tại ô chọn Data Source bên dưới: Chọn `prometheus` > Nhấn nút xanh **Import**.

---

## 4. Hướng dẫn sử dụng các tính năng tương tác (UX/UI)

* **Bộ lọc động (`$container`):**
  * Nhìn lên góc trên bên trái của dashboard `Docker Container Overview`, bạn sẽ thấy thanh chọn `Container`.
  * Bấm vào để lọc xem riêng từng container (ví dụ: chỉ xem `/docker`, `/non-systemd`...) hoặc tích chọn `All` để theo dõi toàn bộ hệ thống.
* **Đọc dải màu cảnh báo (Thresholds):**
  * **Xanh lá cây (Green):** Tài nguyên hoạt động an toàn, bình thường.
  * **Vàng (Yellow):** Tài nguyên bắt đầu tăng cao (> 70% - 75% hoặc chạm ngưỡng cảnh báo).
  * **Đỏ (Red):** Nguy cấp, tài nguyên vượt ngưỡng (> 85% hoặc dịch vụ bị `OFFLINE`).
* **Xem chi tiết đồ thị (Tooltip Mode All):**
  * Chỉ cần rê chuột vào bất kỳ điểm nào trên biểu đồ Time Series, bảng thông số sẽ tự động liệt kê số liệu chi tiết của toàn bộ các đường tại thời điểm đó.

---

## 5. Xử lý lỗi nhanh (Troubleshooting)

| Hiện tượng | Nguyên nhân | Cách khắc phục |
| :--- | :--- | :--- |
| **Không vào được `localhost:3000`** | Container Grafana chưa chạy | Chạy `docker compose up -d grafana` |
| **Panel báo "No data"** | Prometheus chưa kết nối hoặc chưa cào được số liệu | Kiểm tra trang [http://localhost:9090/targets](http://localhost:9090/targets) xem các endpoint đã báo **UP (xanh)** chưa. Vào *Connections > Data Sources* kiểm tra URL là `http://prometheus:9090`. |
| **Bảng cảnh báo Firing rỗng** | Hệ thống đang an toàn, không có lỗi bùng phát | Đây là trạng thái bình thường khi hệ thống khỏe mạnh. |