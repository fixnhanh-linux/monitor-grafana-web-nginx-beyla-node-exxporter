# Monitor Beyla Nginx (Grafana)

Giám sát Nginx bằng **Grafana Beyla** (eBPF, không cần sửa code) + **Node Exporter 1.8.0** (tài nguyên máy chủ), thu thập bằng **Prometheus**, hiển thị trên **Grafana**.

![Sơ đồ kiến trúc giám sát Nginx](docs/so-do-kien-truc.svg)

## Nội dung thư mục

| Tệp | Đặt ở đâu trên server | Mô tả |
|-----|-----------------------|-------|
| `beyla-config.yml` | `/etc/beyla/beyla-config.yml` | Cấu hình Beyla: theo dõi tiến trình `nginx` (cổng 80, 443), xuất metrics HTTP + CPU/RAM tiến trình cho Prometheus |
| `beyla.service` | `/etc/systemd/system/beyla.service` | Systemd service chạy Beyla (quyền root, yêu cầu eBPF) |
| `prometheus-scrape.yml` | Ghép vào `scrape_configs` của `/etc/prometheus/prometheus.yml` | Job `beyla-nginx` và `node_exporter` |
| `grafana-dashboard-beyla-nginx.json` | Grafana > Dashboards > Import | Dashboard "Nginx Beyla + Node Exporter" (Grafana 13.x, định dạng v2), UID `beyla-nginx-node`: HTTP từ Beyla, tài nguyên từ Node Exporter 1.8.0 |
| `Huong-Dan-Giam-Sat-Beyla-Nginx.md` | — | Hướng dẫn cài đặt chi tiết từng bước |
| `prometheus/scrape_configs/*.yml` | `/etc/prometheus/scrape_configs/` | Mỗi site một file (Beyla + Node Exporter), dùng khi Prometheus nạp `scrape_config_files`. Job Beyla phải chứa chữ `beyla`, job Node Exporter phải chứa chữ `node`. Target tài nguyên phải cùng IP với target Beyla |
| `docs/so-do-kien-truc.svg` | — | Sơ đồ kiến trúc (ảnh ở đầu README) |
| `docker/` | Máy chạy Prometheus + Grafana | `docker compose` dựng sẵn Prometheus (scrape Beyla + Node Exporter) và Grafana (đã có datasource) |

## Chạy Prometheus + Grafana bằng Docker

```bash
cd docker
cp .env.example .env        # sửa GRAFANA_ADMIN_PASSWORD
docker compose up -d
```

- Grafana: `http://<máy-docker>:3000`, sau đó Import `grafana-dashboard-beyla-nginx.json`, chọn datasource **Prometheus**.
- Prometheus chỉ mở ở `127.0.0.1:9091` trên máy docker (Grafana gọi nội bộ qua `http://prometheus:9090`).
- Target trong `docker/prometheus/prometheus.yml` đang là `103.75.187.42:9090` (Beyla) và `:9100` (Node Exporter). Trên server Nginx cần mở 2 cổng này **chỉ cho IP máy Prometheus**.
- Sửa `prometheus.yml` xong: `curl -X POST http://127.0.0.1:9091/-/reload`.

## Dashboard gồm

- **HTTP (Beyla):** Request Rate, Throughput, Error Rate 4xx/5xx, Latency p50/p90/p99, p99 theo Route, Top Routes, phân bổ Status Code, Avg Response Size, Client vs Server Duration.
- **Matrix API:** Request Rate theo nhóm endpoint (client / federation / media), Latency p99 Send vs Read Messages.
- **Resources — Overview:** Server Info (Uptime, CPU Cores, RAM Total, Load 1m / Core, số tiến trình nginx) và Resource Usage (gauge % CPU / RAM / Disk / Swap).
- Các hàng **Resources** lặp theo từng máy chủ: chọn nhiều Site / Job thì mỗi máy chủ là một cột cạnh nhau.
- **Resources — CPU & Load:** CPU theo mode (user, system, iowait, steal...), Load Average 1/5/15 so với số core, Processes (running / blocked), Context Switches & Interrupts.
- **Resources — Memory:** RAM & Swap %, Memory chi tiết (Used / Buffers+Cache / Free / Total), Swap I/O.
- **Resources — Disk:** dung lượng theo mount point, Inodes, Disk I/O Utilization, Throughput, IOPS, Disk Latency.
- **Resources — Network:** traffic và packets theo card mạng, Errors & Drops, TCP Connections (established / time_wait / in use), TCP Activity.
- **Resources — Nginx Process:** CPU, RAM của nginx (Beyla, cần `application_process` trong `beyla-config.yml`).
- **Biến lọc:** `DS_PROMETHEUS`, `job`, `instance`, `route` (`server_ip` ẩn, lấy IP từ instance Beyla để ghép với Node Exporter).

## Cài nhanh

1. Cài `beyla` vào `/usr/local/bin/`, copy `beyla-config.yml` và `beyla.service` vào vị trí như bảng trên.
2. `sudo systemctl daemon-reload && sudo systemctl enable --now beyla`
3. Cài Node Exporter (xem Bước 3 trong hướng dẫn).
4. Thêm các job trong `prometheus-scrape.yml` vào Prometheus, rồi `sudo systemctl reload prometheus`.
5. Import `grafana-dashboard-beyla-nginx.json` vào Grafana, chọn datasource Prometheus.

Chi tiết xem [Huong-Dan-Giam-Sat-Beyla-Nginx.md](Huong-Dan-Giam-Sat-Beyla-Nginx.md).

## Lưu ý

- `service.name` / `domain` đang đặt là `scan.lazyy.site` — sửa lại cho đúng site của bạn trong `beyla-config.yml` và `prometheus-scrape.yml`.
- Beyla xuất metrics ở cổng **9090**, trùng cổng mặc định của Prometheus. Nếu Prometheus chạy cùng máy, đổi `prometheus_export.port` (ví dụ `9400`) và sửa target tương ứng trong job `beyla-nginx`.
