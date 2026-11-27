# Runbook - Hướng Dẫn Vận Hành & Xử Lý Sự Cố (N47-Monitoring)

Tài liệu này mô tả quy trình chẩn đoán, xử lý và khắc phục sự cố đối với các cảnh báo (Alerts) trong hệ thống giám sát Prometheus/Grafana của nhóm N47.

---

## 1. Nhóm Cảnh Báo CPU (`HostHighCpuUsage` / `HostCPUHighCritical`)
- **Mức độ:** 
  - *Warning:* CPU >80% liên tục 1 phút (`HostHighCpuUsage`).
  - *Critical:* CPU >90% liên tục 5 phút (`HostCPUHighCritical`).
- **Nguyên nhân phổ biến:** Tiến trình chạy ngầm, vòng lặp vô hạn trong ứng dụng, hoặc quá tải tác vụ xử lý trên server.
- **Quy trình xử lý:**
  1. SSH vào server và chạy lệnh kiểm tra tiến trình ngốn CPU cao nhất:
     ```bash
     top -b -n1 | head -n 20
     # Hoặc sắp xếp theo CPU:
     ps aux --sort=-%cpu | head -n 10
     ```
  2. Xác định PID của tiến trình gây quá tải. Nếu là tiến trình lỗi/treo, tiến hành dừng tiến trình:
     ```bash
     kill -15 <PID>  # Dừng mềm
     kill -9 <PID>   # Buộc dừng (nếu cần)
     ```
  3. Kiểm tra các container Docker liên quan: `docker stats`.

---

## 2. Nhóm Cảnh Báo RAM (`HostMemoryHighWarning` / `HostMemoryHighCritical` / `ContainerMemoryHigh`)
- **Mức độ:** 
  - *Warning:* RAM Host >85% hoặc Container >90%.
  - *Critical:* RAM Host >95% liên tục 5 phút (`HostMemoryHighCritical`). Nguy cơ OOM (Out Of Memory) cao.
- **Nguyên nhân phổ biến:** Rò rỉ bộ nhớ (Memory Leak), phân bổ giới hạn RAM container quá thấp, hoặc các dịch vụ database/cache chiếm hết tài nguyên.
- **Quy trình xử lý:**
  1. Kiểm tra dung lượng RAM thực tế của hệ thống:
     ```bash
     free -h
     ps aux --sort=-%mem | head -n 10
     ```
  2. Nếu nguyên nhân do container vượt giới hạn, kiểm tra giới hạn cấu hình và cập nhật lại RAM cho container:
     ```bash
     docker update --memory="512m" <container_name>
     ```
  3. Khởi động lại dịch vụ hoặc container bị rò rỉ bộ nhớ nếu cần thiết.

---

## 3. Nhóm Cảnh Báo Đĩa (`HostOutOfDiskSpace` / `HostDiskLowCritical`)
- **Mức độ:** 
  - *Warning/Critical:* Dung lượng trống phân vùng `/` dưới 15% (`HostOutOfDiskSpace`).
  - *Critical:* Dung lượng trống dưới 10% liên tục 2 phút (`HostDiskLowCritical`). Cần dừng tác vụ ghi thử nghiệm ngay lập tức.
- **Nguyên nhân phổ biến:** File log hệ thống phình to, dữ liệu database tích tụ, hoặc rác Docker (image/volume cũ chưa dọn).
- **Quy trình xử lý:**
  1. Kiểm tra dung lượng phân vùng đĩa:
     ```bash
     df -h /
     ```
  2. Tìm các thư mục chiếm dung lượng lớn:
     ```bash
     du -sh /* 2>/dev/null | sort -hr
     ```
  3. Dọn dẹp rác Docker để giải phóng dung lượng đĩa lập tức:
     ```bash
     docker system prune -a --volumes -f
     ```
  4. Xóa các file log tạm thời không cần thiết trong `/var/log`.

---

## 4. Nhóm Cảnh Báo Kết Nối (`NodeExporterDown` / `EndpointDown`)
- **Mức độ:** Critical (Mất tín hiệu scrape hoặc probe_success = 0 liên tục trong 1 phút).
- **Nguyên nhân phổ biến:** Container giám sát bị dừng, lỗi mạng nội bộ, hoặc service đích bị sập.
- **Quy trình xử lý:**
  1. Kiểm tra trạng thái các container trong hệ thống:
     ```bash
     docker ps -a
     ```
  2. Kiểm tra log của container gặp sự cố để tìm nguyên nhân:
     ```bash
     docker logs --tail 50 <container_name>
     ```
  3. Khởi động lại container dịch vụ:
     ```bash
     docker restart <container_name>
     ```
