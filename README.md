# Monitor Beyla Nginx (Grafana)

Giám sát Nginx bằng **Grafana Beyla** (eBPF, không cần sửa code) + **Node Exporter** hoặc **Telegraf** (tài nguyên máy chủ), thu thập bằng **Prometheus**, hiển thị trên **Grafana**.

![Sơ đồ kiến trúc giám sát Nginx](docs/so-do-kien-truc.svg)

## Nội dung thư mục

| Tệp | Đặt ở đâu trên server | Mô tả |
|-----|-----------------------|-------|
| `beyla-config.yml` | `/etc/beyla/beyla-config.yml` | Cấu hình Beyla: theo dõi tiến trình `nginx` (cổng 80, 443), xuất metrics HTTP + CPU/RAM tiến trình cho Prometheus |
| `beyla.service` | `/etc/systemd/system/beyla.service` | Systemd service chạy Beyla (quyền root, yêu cầu eBPF) |
| `prometheus-scrape.yml` | Ghép vào `scrape_configs` của `/etc/prometheus/prometheus.yml` | Job `beyla-nginx` và `node_exporter` |
| `grafana-dashboard-beyla-nginx.json` | Grafana > Dashboards > Import | Dashboard (định dạng Grafana v2, Grafana 13.x) |
| `Huong-Dan-Giam-Sat-Beyla-Nginx.md` | — | Hướng dẫn cài đặt chi tiết từng bước |
| `prometheus/scrape_configs/*.yml` | `/etc/prometheus/scrape_configs/` | Mỗi site một file (Beyla + Node Exporter hoặc Telegraf), dùng khi Prometheus nạp `scrape_config_files`. Job Beyla phải chứa chữ `beyla`, job Node Exporter phải chứa chữ `node`. Target tài nguyên phải cùng IP với target Beyla |
| `grafana-dashboard-beyla-nginx-telegraf.json` | Grafana > Dashboards > Import | Bản chỉ dùng **Telegraf** cho phần tài nguyên (không có `node_*`), UID `beyla-nginx-telegraf`. Dùng khi mọi máy chủ đều chạy Telegraf (output `prometheus_client`). Một datasource Prometheus duy nhất (`DS_PROMETHEUS`) phải có cả job Beyla lẫn job Telegraf |
| `telegraf/nginx-access-domain.conf` | `/etc/telegraf/telegraf.d/` trên server nginx | Telegraf đọc access log nginx, xuất `nginx_domain_requests_total` / `nginx_domain_bytes_total` theo `domain` + `resp_code` qua cổng 9125 (cho hàng **Domains** và ô lọc **Domain**) |
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
- **Resources — Overview:** một panel gộp Uptime, CPU Cores, RAM Total, % CPU / RAM / Disk / Swap, Load 1m / Core, số tiến trình nginx.
- Các hàng **Resources** lặp theo từng máy chủ: chọn nhiều Site / Job thì mỗi máy chủ là một cột cạnh nhau.
- **Resources — CPU & Load:** CPU theo mode (user, system, iowait, steal...), Load Average 1/5/15 so với số core.
- **Resources — Memory:** RAM & Swap %, Memory chi tiết (Used / Buffers+Cache / Free / Total).
- **Resources — Disk:** dung lượng theo mount point, Disk I/O Utilization, Throughput, IOPS.
- **Resources — Network:** traffic theo card mạng, Errors & Drops, TCP Connections.
- **Resources — Nginx Process:** CPU, RAM của nginx (Beyla, cần `application_process` trong `beyla-config.yml`) và kết nối nginx stub_status (Telegraf).
- Panel bổ sung (Telegraf, có Node Exporter dự phòng khi có metric tương đương): Processes, Context Switches & Interrupts, Swap I/O, Inodes Used, Disk Latency, Network Packets, TCP Activity, Nginx Connection Rate (accepted / handled / dropped).
- Tài nguyên máy chủ đọc được cả **Node Exporter** lẫn **Telegraf** (output `prometheus_client`): máy chủ có nguồn nào thì hiện nguồn đó. Telegraf không có TIME_WAIT / socket TCP đang mở, nên 2 đường này chỉ có với Node Exporter.
- **Biến lọc:** `DS_PROMETHEUS`, `job`, `instance`, `route`, `domain`, `server_ip`.

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

## Số liệu theo domain (Telegraf đọc access log)

Beyla trên proxy HTTPS/HTTP2 không đọc được tên domain (`server_address` chỉ là tên máy), nên số liệu theo domain lấy từ access log:

```bash
sudo cp telegraf/nginx-access-domain.conf /etc/telegraf/telegraf.d/
sudo usermod -aG adm telegraf          # cho Telegraf đọc /var/log/nginx
sudo systemctl restart telegraf
curl -s localhost:9125/metrics | grep nginx_domain_
```

- Sửa `files` trong file conf theo nơi nginx ghi log. Mỗi domain một file log thì domain lấy theo tên file (`longvan.net.access.log` → `longvan.net`).
- Tất cả domain ghi chung một file: thêm ` host=$host` vào cuối `log_format` của nginx và dùng dòng `grok_patterns` có `host=` (đã ghi sẵn trong file conf).
- Dashboard: hàng **Domains — Access Log (Telegraf)** (Domains Summary, Requests / Bandwidth / 5xx theo domain, Status Codes) và ô lọc **Domain**.
