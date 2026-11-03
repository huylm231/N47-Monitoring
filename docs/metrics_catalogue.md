# DANH MỤC METRICS VÀ PROMQL CHUẨN HÓA

| Mục | Nội dung |
|---|---|
| Dự án | N4.7 – Xây dựng hệ thống giám sát máy chủ, Cloud và trung tâm dữ liệu |
| Người phụ trách | TV3 – Nguyễn Thị Anh Thư (Backend Monitoring) |
| Bàn giao cho | TV4 (Grafana), TV5 (Alert Rules), TV6 (Kiểm thử và KPI) |
| Phiên bản tài liệu | 2.1 |
| Phiên bản phần mềm | Prometheus v3.5.0, Node Exporter v1.9.1, Grafana v12.1.0 |
| Data source Grafana | `http://prometheus:9090` (kết nối nội bộ Docker); `http://localhost:9090` (trình duyệt trên máy host) |
| Trạng thái kiểm tra | Cú pháp toàn bộ truy vấn đã kiểm tra bằng `promtool` 3.5.0. Các truy vấn mục 3 và kiểm tra lưu trữ (TC-04) đã chạy trên Prometheus của nhóm (kết quả ở mục 8, đo với cấu hình Node Exporter gốc). Các biểu thức ngưỡng (mục 5) **chưa chạy** |

---

## 1. Quy ước chung

1. **Tên job:** dùng `node-exporter` cho Node Exporter và `prometheus` cho Prometheus tự giám sát, đúng theo `prometheus.yml`. Không dùng `job="node"` như trong tài liệu Thiết kế kỹ thuật.
2. **Bộ lọc filesystem ảo (bắt buộc cho mọi truy vấn Disk):**

   ```text
   fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"
   ```

   Regex trong PromQL khớp **toàn chuỗi**, không cần `^` và `$`.
3. **Chưa lọc theo `mountpoint`** cho đến khi nhóm dựng xong vùng lưu trữ lab riêng cho kịch bản Disk (tránh trả về rỗng).
4. **Ngưỡng Disk tính theo dung lượng còn trống:** Warning dưới 15%, Critical dưới 10%. Tương đương đã dùng trên 85% và trên 90%.
5. **Tên KPI và tên Alert gọi bằng chữ** (ví dụ "Detection Time", "High CPU Usage") cho đến khi Leader chốt bảng mã duy nhất. **Mã Test Case theo Test Plan.**
6. Truy vấn **không cố định `instance`**, dùng được khi thêm máy mới.
7. Đơn vị Grafana: CPU, RAM, Disk dùng `percent (0-100)`; Network dùng `bytes/sec`; Uptime dùng `seconds`.

---

## 2. Ánh xạ nhóm metric trong tài liệu Phạm vi

| Mã | Metric | Nguồn | Mục truy vấn | Ghi chú |
|---|---|---|---|---|
| MET-01 | CPU Usage | Node Exporter | 3.2 | |
| MET-02 | Memory Usage | Node Exporter | 3.3 | Dùng `MemAvailable` |
| MET-03 | Disk Usage | Node Exporter | 3.4 | Có bộ lọc `fstype` |
| MET-04 | Disk Available | Node Exporter | 3.5 | Tuyệt đối (bytes) và tương đối (%) |
| MET-05 | Network Receive | Node Exporter | 3.6 | |
| MET-06 | Network Transmit | Node Exporter | 3.6 | |
| MET-07 | System Uptime | Node Exporter | 3.7 | |
| MET-08 | Target Availability | Prometheus | 3.1 | Metric `up` |
| MET-09 | Endpoint Availability | Prometheus / Blackbox | 3.1 và 4.3 | Xem lưu ý bên dưới |

**Lưu ý MET-09:** Blackbox Exporter là thành phần tùy chọn và chưa có trong compose. Trong lúc chờ quyết định, trạng thái dịch vụ lấy từ `up` (đúng theo SRS FR-09). `up` chỉ cho biết Prometheus có scrape được target hay không, **không** phải kết quả kiểm tra endpoint nghiệp vụ. Khi bật Blackbox thì dùng `probe_success` (mục 4.3).

---

## 3. Truy vấn chuẩn cho Dashboard (bàn giao TV4)

### 3.1 Target Status (MET-08)

```promql
up
```

Lọc theo job:

```promql
up{job="node-exporter"}
```

Số target đang UP:

```promql
sum(up)
```

Số target DOWN (trả về 0 khi không có target nào down, tránh panel "No data"):

```promql
count(up == 0) or vector(0)
```

- `1` = scrape thành công, `0` = Prometheus không scrape được target.
- Panel gợi ý: Stat hoặc State timeline, mapping `1 = UP`, `0 = DOWN`.

### 3.2 CPU % Usage (MET-01)

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

- Ý nghĩa: 100 trừ phần trăm thời gian CPU ở trạng thái `idle`, trung bình các core.
- `rate()` bắt buộc vì `node_cpu_seconds_total` là counter chỉ tăng dần.

### 3.3 RAM % Usage (MET-02)

```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

RAM khả dụng (GiB):

```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

- Dùng `MemAvailable`, **không** dùng `MemFree`, vì `MemFree` không tính phần cache có thể thu hồi.

### 3.4 Disk % Usage (MET-03)

```promql
(
  1 -
  node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
  /
  node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
) * 100
```

- Mỗi dòng kết quả ứng với một filesystem (nhãn `mountpoint`, `device`, `fstype`).

### 3.5 Disk Available (MET-04)

Dung lượng còn trống (GiB):

```promql
node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"} / 1024 / 1024 / 1024
```

Phần trăm còn trống (dùng cho cảnh báo):

```promql
node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
/
node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
* 100
```

### 3.6 Network RX / TX (MET-05, MET-06)

```promql
rate(node_network_receive_bytes_total{device!="lo"}[5m])
```

```promql
rate(node_network_transmit_bytes_total{device!="lo"}[5m])
```

Bản loại thêm các interface ảo của Docker (tùy chọn, dùng khi kết quả quá nhiều dòng):

```promql
rate(node_network_receive_bytes_total{device!~"lo|veth.*|docker.*|br-.*"}[5m])
```

### 3.7 System Uptime (MET-07)

```promql
node_time_seconds - node_boot_time_seconds
```

Đổi sang ngày:

```promql
(node_time_seconds - node_boot_time_seconds) / 86400
```

- Cách tính này theo tài liệu Thiết kế kỹ thuật mục 6. Kết quả tương đương `time() - node_boot_time_seconds`.

---

## 4. Truy vấn bổ sung

### 4.1 Tải hệ thống và inode (Node Exporter)

Load average 1 phút chia số core (chỉ số tải, không đồng nhất với CPU %):

```promql
node_load1 / on (instance) count by (instance) (node_cpu_seconds_total{mode="idle"})
```

Inode còn trống (%):

```promql
node_filesystem_files_free{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
/
node_filesystem_files{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
* 100
```

### 4.2 Sức khỏe quá trình scrape

Thời gian mỗi lần scrape (giây). Điều tra nếu tăng kéo dài hoặc gần scrape timeout:

```promql
scrape_duration_seconds
```

### 4.3 Thành phần tùy chọn

**Chỉ dùng sau khi nhóm quyết định triển khai và TV2 thêm service vào compose.** Tên nhãn có thể khác theo phiên bản và Docker Desktop, cần kiểm tra bằng Table view.

cAdvisor, CPU theo container (số core trung bình):

```promql
sum by (name) (rate(container_cpu_usage_seconds_total{name!=""}[5m]))
```

cAdvisor, RAM working set theo container (bytes):

```promql
container_memory_working_set_bytes{name!=""}
```

cAdvisor, Network nhận theo container (bytes/s):

```promql
sum by (name) (rate(container_network_receive_bytes_total{name!=""}[5m]))
```

Blackbox, endpoint đang hoạt động (`1`) hay lỗi (`0`):

```promql
probe_success
```

Blackbox, thời gian phản hồi (giây):

```promql
probe_duration_seconds
```

---

## 5. Biểu thức có ngưỡng cho Alert Rules (bàn giao TV5)

TV3 chỉ cung cấp biểu thức. **TV5 đóng gói thành `prometheus/rules/alert_rules.yml`** và cấu hình Alertmanager. Mỗi biểu thức chỉ trả về các series **đang vi phạm**.

| Tên cảnh báo (gợi ý) | Ngưỡng | `for` gợi ý | Mức |
|---|---|---|---|
| High CPU Usage – Warning | CPU trên 80% | 5 phút | Warning |
| High CPU Usage – Critical | CPU trên 90% | 5 phút | Critical |
| High Memory Usage – Warning | RAM trên 85% | 5 phút | Warning |
| High Memory Usage – Critical | RAM trên 95% | 5 phút | Critical |
| Low Disk Space – Warning | Còn trống dưới 15% | 5 phút | Warning |
| Low Disk Space – Critical | Còn trống dưới 10% | 2 phút | Critical |
| Node Exporter Down | `up == 0` | 1 phút | Critical |
| Monitoring Target Down | `up == 0` (mọi job) | 1 phút | Critical |

`for` và ngưỡng lấy từ tài liệu Thiết kế kỹ thuật mục 5.2, là **giá trị khởi điểm**, cần hiệu chỉnh sau khi đo baseline (mục 6.2). Có thể dùng Alertmanager inhibition để Critical thay thế Warning cùng nhóm.

### 5.1 CPU

```promql
(100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 80
```

```promql
(100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 90
```

### 5.2 RAM

```promql
((1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100) > 85
```

```promql
((1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100) > 95
```

### 5.3 Disk (dung lượng còn trống)

```promql
(
  node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
  /
  node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
  * 100
) < 15
```

```promql
(
  node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
  /
  node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs|tracefs|nsfs"}
  * 100
) < 10
```

### 5.4 Target và dịch vụ

```promql
up{job="node-exporter"} == 0
```

```promql
up == 0
```

Khi có Blackbox (tùy chọn):

```promql
probe_success == 0
```

---

## 6. Truy vấn phục vụ đánh giá (bàn giao TV6)

### 6.1 Scrape Success Rate

Tỷ lệ scrape thành công của Node Exporter trong 24 giờ gần nhất (%):

```promql
avg_over_time(up{job="node-exporter"}[24h]) * 100
```

Chỉ có ý nghĩa khi hệ thống đã chạy đủ lâu. Nếu chạy chưa đủ 24 giờ thì đổi cửa sổ cho khớp thời gian thực tế.

### 6.2 Baseline (trung vị, p95, lớn nhất)

Đo trong 1 giờ ở trạng thái bình thường, bước lấy mẫu 1 phút. Ví dụ với CPU:

```promql
quantile_over_time(0.5, (100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))[1h:1m])
```

```promql
quantile_over_time(0.95, (100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))[1h:1m])
```

```promql
max_over_time((100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))[1h:1m])
```

Với RAM, thay biểu thức trong ngoặc bằng công thức mục 3.3. Với Disk, dùng công thức mục 3.4.

### 6.3 Trạng thái cảnh báo (Pending và Firing)

```promql
ALERTS{alertstate="pending"}
```

```promql
ALERTS{alertstate="firing"}
```

Dùng để xác định thời điểm cảnh báo chuyển trạng thái khi tính Detection Time. Thời điểm bắt đầu sự cố do người kiểm thử tự ghi.

### 6.4 Sức khỏe hệ thống giám sát (kiểm tra lưu trữ)

Số time series đang giữ trong bộ nhớ:

```promql
prometheus_tsdb_head_series
```

Dung lượng các block đã lưu trên đĩa (bytes):

```promql
prometheus_tsdb_storage_blocks_bytes
```

RAM Prometheus đang dùng (bytes):

```promql
process_resident_memory_bytes{job="prometheus"}
```

Nếu truy vấn rỗng, kiểm tra tên metric trong ô Metrics explorer của Prometheus UI.

---

## 7. Lưu ý kỹ thuật

1. **Docker Desktop trên Windows:** Node Exporter đo **VM Linux (WSL2) bên trong Docker**, không đo máy Windows. Cần ghi rõ trong README và báo cáo: "hệ thống đo thông số của VM Linux trong Docker Desktop".
2. **Node Exporter chưa mount rootfs:** đã thử `--path.rootfs=/host` cùng volume `/:/host:ro,rslave`, nhưng Docker Desktop báo lỗi `path / is mounted on / but it is not a shared or slave mount` và container không khởi động. Đã quay về cấu hình gốc, nên Disk và Network phản ánh góc nhìn container (interface ảo `eth0`). Phương án thử lại: `/:/host:ro` (bỏ `rslave`), chưa kiểm chứng. Nếu đổi cấu hình, cần chạy lại mục 3.4 và 3.6. Lưu ý: dù mount được rootfs, Node Exporter vẫn chỉ thấy root của VM WSL2, không thấy ổ C: của Windows.
3. **Thư mục `rules/`:** compose đã khai báo mount `./prometheus/rules:/etc/prometheus/rules:ro`. Thư mục này rỗng cho đến khi TV5 thêm `alert_rules.yml`. Nếu thiếu mount, Prometheus vẫn chạy bình thường nhưng **không nạp rule nào và không báo lỗi**.
4. **Retention:** compose đặt `7d`. Tài liệu Thiết kế kỹ thuật đề xuất 15 ngày. Đề xuất chốt 7 ngày cho PoC, chờ Leader xác nhận.
5. **Disk ra 3 series trùng** cho cùng 1 filesystem (`/dev/sdd`) trên Docker Desktop. Khi dùng cho dashboard hoặc alert, bọc bằng `max by (instance, device)(...)` để mỗi đĩa chỉ còn 1 dòng, tránh 3 alert trùng cho 1 đĩa.
6. **Cửa sổ `[5m]`:** hệ thống mới chạy chưa đủ 5 phút có thể cho kết quả rỗng hoặc lệch. Chờ vài phút rồi chạy lại.
7. **Dịch vụ mô phỏng và FR-09:** chưa có người phụ trách. Khi có, TV3 sẽ thêm job scrape hoặc probe tương ứng vào `prometheus.yml`.

---

## 8. Bảng ghi kết quả kiểm thử

Điền sau khi chạy từng truy vấn tại `http://localhost:9090/graph`. Ảnh chụp lưu vào `screenshots/prometheus/`, ghi tên file vào cột "Ảnh chụp". **Không điền kết quả khi chưa chạy.**

**Cấu hình khi đo:** Docker Desktop (Windows, WSL2); Node Exporter v1.9.1 cấu hình gốc, **không** mount rootfs.

| # | Chỉ số | Mục | Test Case (Test Plan) | Kết quả thực tế | Đạt / Không | Ảnh chụp |
|---|---|---|---|---|---|---|
| 1 | Target Status | 3.1 | TC-09 | 2/2 UP | Đạt | 01-targets-up, 02-target-status.png |
| 2 | CPU % | 3.2 | TC-05 | 0.088% | Đạt | 03-cpu-usage.png |
| 3 | RAM % | 3.3 | TC-06 | 11.70% | Đạt | 04-ram-usage.png |
| 4 | Disk % (có lọc) | 3.4 | TC-07 | 7.6074% – /dev/sdd (ext4), đĩa ảo VM WSL2, hiện qua 3 mountpoint bind-mount trùng nhau | Đạt | 05-disk-usage.png |
| 5 | Disk Available | 3.5 | TC-07 | 930.26 GiB (92.39% trống) – /dev/sdd, đĩa ảo VM WSL2 | Đạt | 06-disk-available.png |
| 6 | Network RX | 3.6 | TC-08 | 42.00 bytes/s – eth0 (container) | Có dữ liệu | 07-network-rx.png  |
| 7 | Network TX | 3.6 | TC-08 | 821.80 bytes/s – eth0 (container) | Có dữ liệu | 08-network-tx.png |
| 8 | Uptime | 3.7 | Chưa có TC riêng | 10,839.40 giây | Có dữ liệu | 09-system-uptime.png |
| 9 | Dữ liệu còn sau khi restart container | 6.4 | TC-04 | Trước: 55 mẫu (17:00, giờ máy UTC+7) → Sau: 76 mẫu (17:02, giờ máy UTC+7). Truy vấn offset 10m vẫn trả về 1; graph 1h liên tục, dữ liệu trước restart còn nguyên | Đạt | 10-tc04-before-restart.png, 11-tc04-after-restart.png, 12-tc04-graph-1h.png, 13-tc04-targets-up.png |
| 10 | Biểu thức CPU có ngưỡng | 5.1 | Kiểm thử bởi TV5 | | | |
| 11 | Biểu thức RAM có ngưỡng | 5.2 | Kiểm thử bởi TV5 | | | |
| 12 | Biểu thức Disk có ngưỡng | 5.3 | Kiểm thử bởi TV5 | | | |
| 13 | Biểu thức Target Down | 5.4 | Kiểm thử bởi TV5 | | | |

**Ghi chú cho các dòng 4 đến 8:**

- Disk (dòng 4, 5): đã xác minh. Cả 3 series cùng `/dev/sdd` (ext4), là đĩa ảo của VM WSL2, hiện qua 3 bind-mount của Docker (`/etc/hostname`, `/etc/hosts`, `/etc/resolv.conf`). Không phải ổ đĩa Windows.
- Network (dòng 6, 7): đã xác minh. Chỉ có `eth0` (lưu lượng container), không phải card mạng của máy.
- Uptime (dòng 8): là thời gian chạy của VM Linux trong Docker Desktop (khoảng 3 giờ), không phải thời gian bật máy Windows.
- Múi giờ: giờ ghi ở dòng 9 là giờ máy (UTC+7). Trục thời gian trên Prometheus UI hiển thị theo UTC nên ảnh graph lệch 7 giờ (ví dụ 10:35 trên graph = 17:35 giờ máy).
- TC-04 (dòng 9): dùng `count_over_time(up{job="node-exporter"}[1h])` và `up{job="node-exporter"} offset 10m`, restart bằng `docker compose restart prometheus` (không dùng `down -v`). Số series trong bộ nhớ trước restart: 1487 (prometheus_tsdb_head_series). Ảnh 13 chụp sau khi kiểm thử.
- Nếu sau này đổi cấu hình Node Exporter (xem mục 7.2) thì phải đo lại dòng 4 đến 8.

---

## 9. Các điểm đang chờ Leader quyết định

Chi tiết xem `docs/tong-hop-bat-dong-tv3.md`. Tài liệu này sẽ cập nhật khi có quyết định.

| Điểm | Tác động lên tài liệu này |
|---|---|
| Môi trường chính (Windows hay Linux VM) | Mục 7.1, cách đo Disk và Network |
| Cấu hình rootfs của Node Exporter (TV2) | Mục 3.4, 3.6 |
| Có triển khai Blackbox và cAdvisor hay không | Mục 4.3, MET-09 |
| Người phụ trách FR-09 và dịch vụ mô phỏng | Mục 7.7 |
| Bảng mã KPI và Alert thống nhất | Mục 5, 6 |
| Thời gian lưu trữ (7 hay 15 ngày) | Mục 7.4 |

---

## 10. Lịch sử thay đổi

| Phiên bản | Nội dung |
|---|---|
| 1.0 | Bản đầu: CPU, RAM, Disk, Network, Uptime, Target Status và biểu thức ngưỡng cơ bản |
| 2.0 | Viết lại có dấu; bổ sung ánh xạ MET-01 đến MET-09, ngưỡng Critical, Disk Available, load, inode, cAdvisor và Blackbox (tùy chọn), truy vấn đánh giá cho TV6, baseline, trạng thái cảnh báo, kiểm tra lưu trữ; chuẩn hóa ngưỡng Disk theo dung lượng còn trống; bỏ mẫu `alert_rules.yml` (thuộc TV5); đổi cách gọi KPI và Alert theo tên |
| 2.1 | Điền kết quả đo mục 3 (TC-05 đến TC-08); ghi chú giới hạn Disk, Network, Uptime; cập nhật mục 7.2 và 7.3 theo cấu hình compose hiện tại; xác minh đối tượng đo Disk và Network (/dev/sdd, eth0); bổ sung ảnh chụp; hoàn thành TC-04 (dữ liệu còn sau restart) |
