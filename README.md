# N47-Monitoring

# N4.7 – Server Monitoring System

Hệ thống giám sát máy chủ và dịch vụ được triển khai bằng Docker Compose, sử dụng Prometheus để thu thập và lưu trữ metrics, Node Exporter để cung cấp chỉ số hệ thống và Grafana để trực quan hóa dữ liệu.

## 1. Mục tiêu

- Triển khai môi trường giám sát bằng Docker Compose.
- Thu thập metrics hệ thống thông qua Node Exporter.
- Sử dụng Prometheus để thu thập và truy vấn metrics.
- Sử dụng Grafana để xây dựng dashboard giám sát.
- Làm nền tảng phát triển cảnh báo và kiểm thử sự cố ở các giai đoạn tiếp theo.

## 2. Công nghệ sử dụng

| Công nghệ      | Vai trò                             |
| -------------- | ----------------------------------- |
| Docker Desktop | Chạy các container                  |
| Docker Compose | Quản lý và khởi chạy các dịch vụ    |
| Prometheus     | Thu thập và lưu trữ metrics         |
| Node Exporter  | Cung cấp metrics hệ thống           |
| Grafana        | Trực quan hóa metrics qua dashboard |

## 3. Kiến trúc hệ thống

```text
Node Exporter
     |
     | System Metrics
     v
 Prometheus
     |
     | PromQL
     v
  Grafana
     |
     v
 Monitoring Dashboard
```

Các dịch vụ được kết nối trong mạng Docker Compose nội bộ.

## 4. Yêu cầu môi trường

Trước khi triển khai, cần cài đặt:

- Docker Desktop.(Bản mới nhất)
- Docker Compose V2.
- PowerShell hoặc terminal trong VS Code.
- Trình duyệt web hiện đại.

Kiểm tra Docker:

```powershell
docker --version
docker compose version
docker info
```

Đảm bảo Docker Desktop đã khởi động và Docker Engine đang hoạt động.

## 5. Cấu trúc thư mục

```text
N47-Monitoring/
├── docker-compose.yml
├── .gitignore
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   └── dashboards/
│   └── dashboards/
├── docs/
└── tests/
```

Ý nghĩa các thư mục:

- `docker-compose.yml`: khai báo các dịch vụ Docker.
- `prometheus/prometheus.yml`: cấu hình thu thập metrics.
- `prometheus/rules/`: lưu các quy tắc cảnh báo khi được triển khai.
- `grafana/provisioning/`: cấu hình tự động nguồn dữ liệu và dashboard khi được bổ sung.
- `grafana/dashboards/`: lưu cấu hình dashboard.
- `docs/`: tài liệu kỹ thuật và hướng dẫn vận hành.
- `tests/`: kịch bản và kết quả kiểm thử.

## 6. Triển khai hệ thống

### Bước 1: Mở thư mục dự án

Mở PowerShell và chạy:

```powershell
cd D:\N47-Monitoring
```

Kiểm tra file cấu hình:

```powershell
dir
```

Đảm bảo thư mục hiện tại có file `docker-compose.yml`.

### Bước 2: Kiểm tra cấu hình

```powershell
docker compose config
```

Nếu cấu hình hợp lệ, Docker Compose sẽ hiển thị cấu hình đã được phân tích.

### Bước 3: Khởi chạy các dịch vụ

```powershell
docker compose up -d
```

Lần triển khai đầu tiên có thể mất thời gian vì Docker cần tải các image.

### Bước 4: Kiểm tra trạng thái container

```powershell
docker compose ps
```

Các container cần có:

| Container           | Cổng | Chức năng                    |
| ------------------- | ---: | ---------------------------- |
| `n47-prometheus`    | 9090 | Thu thập và truy vấn metrics |
| `n47-node-exporter` | 9100 | Cung cấp metrics hệ thống    |
| `n47-grafana`       | 3000 | Dashboard giám sát           |

Trạng thái `Up` cho biết container đang chạy. Cần kiểm tra thêm endpoint và trạng thái target để xác nhận hệ thống giám sát hoạt động bình thường.

### Bước 5: Kiểm tra log khi có lỗi

Xem log của toàn bộ dịch vụ:

```powershell
docker compose logs --tail=100
```

Xem log riêng từng dịch vụ:

```powershell
docker compose logs prometheus
docker compose logs node-exporter
docker compose logs grafana
```

## 7. Truy cập hệ thống

Sau khi các container khởi động thành công, truy cập các địa chỉ sau:

| Dịch vụ            | URL                           |
| ------------------ | ----------------------------- |
| Prometheus         | http://localhost:9090         |
| Prometheus Targets | http://localhost:9090/targets |
| Prometheus Graph   | http://localhost:9090/graph   |
| Node Exporter      | http://localhost:9100/metrics |
| Grafana            | http://localhost:3000         |

### Đăng nhập Grafana

Theo cấu hình lab hiện tại:

- Username: `admin`
- Password: `admin12345`

Đây là thông tin đăng nhập dành cho môi trường thử nghiệm. Không sử dụng mật khẩu này trong môi trường production.

## 8. Cấu hình Prometheus trong Grafana

Để Grafana truy vấn được dữ liệu từ Prometheus:

1. Đăng nhập Grafana.
2. Mở **Connections → Data sources**.
3. Chọn **Add data source → Prometheus**.
4. Nhập URL:

```text
http://prometheus:9090
```

5. Chọn **Save & test**.

Sử dụng tên dịch vụ `prometheus` vì Grafana truy cập Prometheus thông qua mạng nội bộ Docker Compose.

## 9. Kiểm tra metrics

Mở http://localhost:9090/graph và chạy các truy vấn PromQL sau.

### 9.1. Kiểm tra trạng thái thu thập dữ liệu

```promql
up
```

Giá trị `1` biểu thị target đang được Prometheus scrape thành công; `0` biểu thị scrape thất bại.

### 9.2. Kiểm tra CPU

```promql
node_cpu_seconds_total
```

Truy vấn này hiển thị bộ đếm thời gian CPU theo các chế độ hoạt động.

Tính mức sử dụng CPU trung bình theo instance:

```promql
100 * (1 - avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
))
```

### 9.3. Kiểm tra RAM

```promql
node_memory_MemAvailable_bytes
```

Tính phần trăm RAM đang sử dụng theo metric khả dụng:

```promql
100 * (
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

### 9.4. Kiểm tra dung lượng ổ đĩa

```promql
node_filesystem_avail_bytes
```

### 9.5. Kiểm tra lưu lượng mạng

```promql
rate(node_network_receive_bytes_total[5m])
```

Các metric chỉ xuất hiện khi target tương ứng hoạt động và metric đó được exporter cung cấp.

## 10. Các lệnh vận hành thường dùng

### Khởi chạy dịch vụ

```powershell
docker compose up -d
```

### Kiểm tra trạng thái

```powershell
docker compose ps
```

### Xem log

```powershell
docker compose logs -f
```

Nhấn `Ctrl + C` để ngừng theo dõi log. Thao tác này không dừng các container.

### Khởi động lại dịch vụ

```powershell
docker compose restart
```

### Dừng và xóa container, network do Compose tạo

```powershell
docker compose down
```

Lệnh trên giữ lại các named volume của Prometheus và Grafana.

**Không chạy `docker compose down -v` nếu muốn giữ dữ liệu metrics và dữ liệu Grafana**, vì tùy chọn `-v` sẽ xóa các named volume do Compose quản lý.

## 11. Xử lý sự cố

### Docker không tìm thấy file Compose

Thông báo:

```text
no configuration file provided: not found
```

Cách xử lý:

```powershell
cd D:\N47-Monitoring
dir
```

Kiểm tra file có tên chính xác `docker-compose.yml` hoặc `compose.yaml`.

### Lỗi tải image hoặc `EOF`

Thông báo có thể chứa:

```text
failed to copy: httpReadSeeker: failed open
```

Cách xử lý:

1. Kiểm tra kết nối Internet.
2. Đảm bảo Docker Desktop đang chạy.
3. Thử tải lại image:

```powershell
docker pull prom/prometheus:v3.5.0
docker pull prom/node-exporter:v1.9.1
docker pull grafana/grafana:12.1.0
```

4. Nếu vẫn lỗi, thử mạng khác hoặc kiểm tra VPN/proxy và kết nối tới máy chủ tải image.

### Prometheus target ở trạng thái DOWN

Kiểm tra cấu hình trong `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

Sau khi chỉnh sửa cấu hình, tải lại dịch vụ Prometheus:

```powershell
docker compose restart prometheus
```

Sau đó kiểm tra lại http://localhost:9090/targets.

### Grafana không kết nối được Prometheus

Kiểm tra URL nguồn dữ liệu:

```text
http://prometheus:9090
```

Không dùng `localhost:9090` trong cấu hình nguồn dữ liệu của Grafana khi cả hai đang chạy trong các container riêng biệt.
