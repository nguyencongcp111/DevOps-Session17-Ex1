# Bài tập: Thiết lập bộ ba Elasticsearch, Fluentd và Kibana (EFK)

**Ngày thực hiện:** 06/11/2026
**Dự án:** `order-app` (Spring Boot 3.3, Java 17, PostgreSQL)
**Môi trường chạy:** AWS EC2 (Ubuntu, khoảng 2GB RAM), toàn bộ chạy bằng Docker Compose

---

## 1. Cấu hình gốc và vấn đề

### Tình huống

Dev team quyết định dùng EFK (ELK) để quản lý log. Một bạn Dev đã viết file `docker-compose.yml` nhưng khi chạy lên, Kibana liên tục báo lỗi không kết nối được Elasticsearch (`Elasticsearch is unreachable`, giao diện hiện `Kibana server is not ready yet`).

### Cấu hình gốc

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    environment:
      - discovery.type=single-node
  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9300
```

### Vấn đề

Elasticsearch mở hai port với hai mục đích khác nhau:

| Port | Giao thức | Dùng để |
|---|---|---|
| `9200` | HTTP (REST API) | Client, Kibana, Fluentd giao tiếp với Elasticsearch |
| `9300` | Transport (nhị phân) | Các node Elasticsearch giao tiếp với nhau |

Kibana chỉ biết nói HTTP, nhưng biến `ELASTICSEARCH_HOSTS` lại trỏ vào port **9300** (transport). Vì vậy Kibana không thể lấy được thông tin phiên bản của Elasticsearch và không khởi động hoàn chỉnh được.

---

## 2. Yêu cầu và cách thực hiện

### Yêu cầu

1. Sửa biến `ELASTICSEARCH_HOSTS` của Kibana về đúng port HTTP của Elasticsearch (`9200`).
2. Chạy lại hệ thống và truy cập giao diện Kibana.

### Bước 1. Chuẩn bị hệ thống để `order-app` đẩy log vào EFK

Luồng log:

```
order-app (JSON) -> logs/app.log -> Fluentd (đọc file) -> Elasticsearch -> Kibana
```

Ứng dụng vẫn ghi log JSON vào `logs/app.log` bằng `LogstashEncoder` như các bài trước nên không phải sửa code. Các thành phần thêm vào:

- `Dockerfile` cho `order-app`: build bằng Maven rồi chạy bằng JRE 17.
- `fluentd/Dockerfile`: image Fluentd có cài plugin `fluent-plugin-elasticsearch`.
- `fluentd/fluent.conf`: đọc `app.log`, parse JSON, gửi sang Elasticsearch.
- `docker-compose.yml`: gồm 5 service là `postgres`, `order-app`, `fluentd`, `elasticsearch`, `kibana`.

Cấu hình Fluentd (`fluentd/fluent.conf`):

```
<source>
  @type tail
  path /app-logs/app.log
  pos_file /tmp/app.log.pos
  tag order.app
  read_from_head true
  <parse>
    @type json
  </parse>
</source>

<match order.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix order-service
  <buffer>
    flush_interval 5s
  </buffer>
</match>
```

Các điểm chính trong `docker-compose.yml`:

- Elasticsearch và Kibana chỉ bind vào `127.0.0.1` (`127.0.0.1:9200:9200`, `127.0.0.1:5601:5601`), vì Elasticsearch 7.17 mặc định không có xác thực, không nên mở ra Internet.
- Elasticsearch đặt `ES_JAVA_OPTS=-Xms512m -Xmx512m` để phù hợp máy nhỏ.
- `order-app` chờ Postgres `healthy` rồi mới chạy (`depends_on` kèm `healthcheck`).
- Port Postgres trên host là `5433`, vì `5432` đã bị container của project khác chiếm.

Phần Kibana được giữ nguyên lỗi của đề để tái hiện:

```yaml
kibana:
  image: docker.elastic.co/kibana/kibana:7.17.0
  environment:
    - ELASTICSEARCH_HOSTS=http://elasticsearch:9300   # lỗi của đề
```

```bash
docker compose up -d --build
```

### Bước 2. Tái hiện lỗi

```bash
docker compose logs kibana | tail -20
curl -s localhost:5601/api/status | head -c 300
```

Log của Kibana ghi nhận hai lỗi:

```text
Unable to retrieve version information from Elasticsearch nodes. connect ECONNREFUSED 172.18.0.2:9300
Unable to retrieve version information from Elasticsearch nodes. Parse Error: Expected HTTP/
```

- `ECONNREFUSED`: lúc đầu Elasticsearch chưa mở port 9300 (đang khởi động).
- `Expected HTTP/`: khi port 9300 mở, Kibana gửi HTTP vào nhưng nhận lại dữ liệu nhị phân của giao thức transport nên không đọc được.

### Bước 3. Sửa cấu hình và chạy lại

```bash
sed -i 's#http://elasticsearch:9300#http://elasticsearch:9200#' docker-compose.yml
sed -i '/Lỗi của đề/d' docker-compose.yml
grep -n ELASTICSEARCH_HOSTS docker-compose.yml
docker compose up -d kibana
```

Cấu hình sau khi sửa:

```yaml
environment:
  - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
```

### Bước 4. Truy cập Kibana và tạo index pattern

Mở SSH tunnel từ máy cá nhân (không phải mở Security Group):

```bash
ssh -i /duong/dan/key.pem -L 5601:localhost:5601 ubuntu@IP_EC2
```

Sau đó vào `http://localhost:5601`, rồi:

1. **Stack Management**, **Index Patterns**, **Create index pattern**.
2. Nhập `order-service-*`, chọn time field `@timestamp`.
3. Vào **Discover**, lọc bằng KQL, ví dụ `level: "ERROR"`.

---

## 3. Kết quả

### Kibana hoạt động bình thường ở cổng 5601

Sau khi sửa về port `9200`, `GET /api/status` trả về trạng thái `green`, và giao diện Kibana hiển thị đầy đủ (không còn `Kibana server is not ready yet`):

![Trang chủ Kibana tại localhost:5601](images/kibana-home.png)

### Log của order-app hiển thị trong Discover

Fluentd đẩy log vào index `order-service-2026.11.06`. Trong Discover, truy vấn `level: "ERROR"` trả về 2 kết quả, đúng với 2 lần gọi endpoint `/api/debug/error`. Mỗi log là một document có đủ các field riêng: `level`, `app_name`, `env`, `logger_name`, `message`, `thread_name`, `stack_trace`.

![Discover hiển thị log ERROR của order-service](images/kibana-discover.png)

### Nhận xét

- Chỉ cần đổi `9300` thành `9200` là Kibana kết nối được Elasticsearch. Nguyên tắc: mọi client dùng REST API (Kibana, Fluentd, `curl`) đều dùng `9200`; `9300` chỉ dành cho giao tiếp nội bộ giữa các node Elasticsearch.
- Dữ liệu vẫn vào Elasticsearch trong lúc Kibana bị lỗi, vì Fluentd cấu hình đúng port `9200`. Lỗi nằm riêng ở Kibana.
- Nhờ làm các bài trước (log JSON, `app_name`, `env`, `stack_trace` trong một field), log hiển thị trong Kibana đã có cấu trúc. Không cần cấu hình multiline hay grok, và lỗi Exception nhiều dòng nằm gọn trong một document.
- Có thể lọc trực tiếp trên Kibana, ví dụ `app_name: "order-service" and env: "production" and level: "ERROR"`.

### Sự cố phát sinh: Elasticsearch khóa index vì đầy ổ đĩa

Sau khi sửa port, log Kibana vẫn báo lỗi khác:

```text
index [.kibana_task_manager_7.17.0_001] blocked by: [TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark, index has read-only-allow-delete block]
```

- **Nguyên nhân:** ổ đĩa EC2 19GB đã dùng 99% (còn 201MB). Khi dung lượng vượt 95% (ngưỡng flood-stage), Elasticsearch tự chuyển các index sang chế độ chỉ đọc để tránh hỏng dữ liệu. Docker chiếm khoảng 13GB (images 10GB, build cache 3GB).
- **Cách xử lý:** dọn build cache và image không dùng, không động vào volume chứa dữ liệu:

```bash
docker builder prune -af
docker image prune -f
df -h /
```

- **Kết quả:** dung lượng xuống còn 57% (trống 8.1GB). Khóa chỉ đọc tự gỡ khi ổ đĩa xuống dưới ngưỡng 90%. Sau khi `docker compose restart kibana`, `/api/status` báo `green`, và số document trong index tăng từ 23 lên 29, chứng tỏ Fluentd vẫn ghi dữ liệu bình thường.
- Sự cố này cùng bản chất với vấn đề của chuỗi bài về log: dữ liệu log và image không được kiểm soát sẽ làm đầy ổ đĩa. Với hệ thống thật cần giám sát dung lượng, dùng ILM (Index Lifecycle Management) để xóa index cũ, và cấp ổ đĩa đủ lớn.

### Tài nguyên máy

Máy chỉ có khoảng 1.9GB RAM, toàn stack dùng khoảng 1.5GB và có dùng swap. Đủ để làm bài, nhưng với hệ thống thật nên dùng máy từ 4GB RAM trở lên (ví dụ t3.medium).

### Hướng mở rộng

- Đặt `number_of_replicas: 0` cho môi trường một node để index luôn `green` (mặc định là `yellow` do không có node để đặt replica).
- Thêm ILM hoặc job xóa index cũ (`order-service-*`) sau N ngày.
- Bật xác thực (`xpack.security`) trước khi mở Elasticsearch/Kibana ra ngoài mạng.
- Tạo dashboard trên Kibana thống kê số lỗi theo `app_name`, `env` và theo thời gian.
