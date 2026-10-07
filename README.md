# Bài 3: Thiết lập Cơ sở dữ liệu và Tự cấu hình dịch vụ Systemd cho Spring Boot

## 1. Mục tiêu

- Khởi tạo cơ sở dữ liệu **MySQL** (`springboot_db`) và tài khoản ứng dụng có **phân quyền bảo mật**
  (chỉ toàn quyền trên đúng database của mình).
- **Tự viết** tệp cấu hình dịch vụ Systemd `/etc/systemd/system/spring-app.service` từ đầu
  (không dùng tệp mẫu) để quản lý vòng đời ứng dụng Spring Boot.
- Chạy ứng dụng dưới tài khoản **non-root** (`spring-runner`, không có shell đăng nhập) để bảo vệ hệ điều hành.
- Ứng dụng lắng nghe cổng **8082**, tự động khởi động lại sau **10 giây** nếu bị crash bất ngờ.

## 2. Bối cảnh & ràng buộc của đề bài

| Ràng buộc | Thực hiện |
|---|---|
| Tạo database `springboot_db` | ✅ (`CREATE DATABASE`) |
| Tạo user `spring-admin` / `SpringSecure@123`, toàn quyền trên `springboot_db` | ✅ (`GRANT ALL PRIVILEGES ON springboot_db.*`) |
| User hệ thống Linux `spring-runner`, **không có shell login** | ✅ (`useradd -r -s /usr/sbin/nologin`) |
| Tệp unit `/etc/systemd/system/spring-app.service` tự viết | ✅ (xem [`spring-app.service`](./spring-app.service)) |
| Chạy dưới user `spring-runner` | ✅ (`User=` / `Group=`) |
| `ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar` | ✅ |
| Tự khởi động lại sau 10 giây khi crash | ✅ (`Restart=on-failure`, `RestartSec=10`) |
| Lắng nghe cổng 8082 | ✅ (xác nhận bằng `ss -tlnp`) |

## 3. Môi trường thực hiện

| Thành phần | Giá trị |
|---|---|
| Hệ điều hành | Ubuntu 24.04.4 LTS, kernel `6.17.0-1022-azure` |
| Init / service manager | `systemd 255` (PID 1) |
| MySQL | `mysqld  Ver 8.0.46-0ubuntu0.24.04.4` |
| Java runtime | OpenJDK `21.0.12.1` (`/usr/bin/java`) |
| Tài khoản dịch vụ | `spring-runner` (uid=997, gid=987), shell `/usr/sbin/nologin` |
| Thư mục ứng dụng | `/opt/spring-app/` (chứa `app.jar`, `application.properties`, JDBC driver) |
| Tệp unit | `/etc/systemd/system/spring-app.service` |

> **Lưu ý về `app.jar`:** đề bài cấp sẵn tệp `app.jar` mẫu. Môi trường lab không có sẵn artifact này,
> nên `app.jar` được **tạo bằng một launcher mô phỏng Spring Boot** (xem [`app/`](./app/) — mã nguồn đầy đủ).
> Launcher này chạy thật qua đúng lệnh `java -jar /opt/spring-app/app.jar`, **mở socket HTTP cổng 8082**
> và **kết nối MySQL qua JDBC** (thật) bằng tài khoản `spring-admin`. Khi có JAR Spring Boot thật,
> chỉ cần **thay thế nội dung `/opt/spring-app/app.jar`** — tệp `spring-app.service` giữ nguyên không đổi.

## 4. Cấu trúc tệp nộp bài

```
homework/session_07/ex3/
├── README.md                     # báo cáo này
├── spring-app.service            # tệp cấu hình Systemd tự viết (đặt tại /etc/systemd/system/)
├── app/
│   ├── SpringAppApplication.java # launcher Spring Boot mô phỏng (bind 8082, nối JDBC MySQL)
│   ├── application.properties    # cấu hình spring.datasource.* trỏ tới springboot_db
│   ├── MANIFEST.MF               # Main-Class + Class-Path tới JDBC driver
│   └── build.sh                  # biên dịch app.jar và triển khai vào /opt/spring-app
└── db/
    └── setup.sql                 # script khởi tạo database + user + phân quyền
```

## 5. Các bước thực hiện

### 5.1. Khởi tạo database `springboot_db` và user `spring-admin`

[`db/setup.sql`](./db/setup.sql):

```sql
CREATE DATABASE IF NOT EXISTS springboot_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER IF NOT EXISTS 'spring-admin'@'localhost'
  IDENTIFIED BY 'SpringSecure@123';

GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'localhost';

FLUSH PRIVILEGES;
```

Thực thi (MySQL 8 dùng `auth_socket` cho `root` nên chạy được bằng `sudo mysql`):

```bash
sudo mysql < db/setup.sql
sudo mysql -e "SHOW DATABASES LIKE 'springboot_db'; SELECT user,host,plugin FROM mysql.user WHERE user='spring-admin'; SHOW GRANTS FOR 'spring-admin'@'localhost';"
```

```console
$ sudo mysql -e "..."
Database (springboot_db)
springboot_db
user           host       plugin
spring-admin   localhost  caching_sha2_password

Grants for spring-admin@localhost
GRANT USAGE ON *.* TO `spring-admin`@`localhost`
GRANT ALL PRIVILEGES ON `springboot_db`.* TO `spring-admin`@`localhost`
```

Kiểm chứng đăng nhập bằng chính tài khoản ứng dụng và **giới hạn quyền** đúng database:

```console
$ mysql -h 127.0.0.1 -u spring-admin -p'SpringSecure@123' springboot_db -e "SELECT DATABASE(); SELECT VERSION();"
mysql: [Warning] Using a password on the command line interface can be insecure.
DATABASE()
springboot_db
VERSION()
8.0.46-0ubuntu0.24.04.4
```

> **Bảo mật:** `spring-admin` chỉ được `GRANT ALL PRIVILEGES` trên **đúng** `springboot_db.*` —
> ngoài database này user chỉ có `USAGE` (không quyền). Password nằm trong `application.properties`
> được đặt quyền `0640 root:spring-runner` (chỉ `root` và user chạy ứng dụng đọc được).

### 5.2. Tạo user hệ thống `spring-runner` (không login)

```bash
sudo useradd -r -s /usr/sbin/nologin spring-runner
```

Cờ `-r` tạo **system account** (không có home, không dùng cho đăng nhập người); `-s /usr/sbin/nologin`
chặn mọi phiên shell.

```console
$ id spring-runner
uid=997(spring-runner) gid=987(spring-runner) groups=987(spring-runner)

$ getent passwd spring-runner
spring-runner:x:997:987::/home/spring-runner:/usr/sbin/nologin

$ su - spring-runner
su: warning: cannot change directory to /home/spring-runner: No such file or directory
This account is currently not available.        # /usr/sbin/nologin chặn đăng nhập
```

### 5.3. Triển khai ứng dụng vào `/opt/spring-app`

Cấu hình kết nối DB nằm trong [`app/application.properties`](./app/application.properties):

```properties
server.port=8082
spring.application.name=spring-app

# MySQL datasource -> database springboot_db, user spring-admin
spring.datasource.url=jdbc:mysql://localhost:3306/springboot_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
spring.datasource.username=spring-admin
spring.datasource.password=SpringSecure@123
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```

Biên dịch & triển khai bằng [`app/build.sh`](./app/build.sh):

```bash
cd app && ./build.sh
```

```console
$ ls -l /opt/spring-app
-rw-r--r-- 1 root root           app.jar
-rw-r----- 1 root spring-runner  application.properties
-rw-r--r-- 1 root root           mysql-connector-j-9.1.0.jar
```

`app.jar` là **executable jar** (`Main-Class: SpringAppApplication`) và tham chiếu JDBC driver qua
`Class-Path: mysql-connector-j-9.1.0.jar` trong manifest — nên lệnh `java -jar /opt/spring-app/app.jar`
vẫn nạp được driver MySQL mà không cần thêm `-cp`.

### 5.4. Tệp cấu hình Systemd tự viết — [`spring-app.service`](./spring-app.service)

Đặt tại `/etc/systemd/system/spring-app.service`:

```ini
[Unit]
Description=Spring Boot Application (spring-app)
Documentation=file:/opt/spring-app/app.jar
After=network-online.target mysql.service
Wants=network-online.target mysql.service

[Service]
Type=simple
User=spring-runner
Group=spring-runner
WorkingDirectory=/opt/spring-app
ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Giải thích các directive:

| Block | Directive | Ý nghĩa |
|---|---|---|
| `[Unit]` | `Description` | Mô tả ngắn hiển thị trong `systemctl status`. |
| `[Unit]` | `Documentation` | Trỏ tới artifact để tra cứu (`systemctl status` in ra mục `Docs:`). |
| `[Unit]` | `After=network-online.target mysql.service` | Chỉ khởi động **sau khi** mạng sẵn sàng và **sau** MySQL (ứng dụng cần DB). |
| `[Unit]` | `Wants=network-online.target mysql.service` | Kéo hai target/service này vào khi service được start. |
| `[Service]` | `Type=simple` | Tiến trình chính chính là `ExecStart`, chạy **foreground** (đúng với JVM). |
| `[Service]` | `User` / `Group` | Chạy dưới `spring-runner` (**non-root**). |
| `[Service]` | `WorkingDirectory` | Thư mục làm việc của tiến trình (`/opt/spring-app`). |
| `[Service]` | `ExecStart` | Lệnh khởi chạy: `/usr/bin/java -jar /opt/spring-app/app.jar`. |
| `[Service]` | `SuccessExitStatus=143` | Coi exit code `143` (128+15, do JVM nhận `SIGTERM` khi `stop`) là **thoát thành công** → `systemctl stop` kết thúc ở `inactive` sạch sẽ thay vì `failed`. |
| `[Service]` | `Restart=on-failure` | **Chỉ** restart khi thoát *bất thường* (exit code ≠ 0 hoặc bị kill bởi signal). Thoát sạch sẽ không restart. |
| `[Service]` | `RestartSec=10` | Chờ **10 giây** trước khi khởi động lại. |
| `[Install]` | `WantedBy=multi-user.target` | Bật dịch vụ ở chế độ đa người dùng → **tự chạy khi boot**. |

### 5.5. Kích hoạt dịch vụ

> Bắt buộc `daemon-reload` sau khi tạo/sửa tệp `.service` để systemd đọc lại cấu hình.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spring-app.service
```

```console
$ sudo systemctl enable --now spring-app.service
Created symlink /etc/systemd/system/multi-user.target.wants/spring-app.service → /etc/systemd/system/spring-app.service.
```

## 6. Kiểm tra kết quả

### 6.1. `sudo systemctl status spring-app.service` → `active (running)`

```console
$ sudo systemctl status spring-app.service
● spring-app.service - Spring Boot Application (spring-app)
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-10-07 08:26:27 UTC; 6s ago
       Docs: file:/opt/spring-app/app.jar
   Main PID: 214549 (java)
      Tasks: 20 (limit: 4600)
     Memory: 42.3M (peak: 43.0M)
        CPU: 863ms
     CGroup: /system.slice/spring-app.service
             └─214549 /usr/bin/java -jar /opt/spring-app/app.jar

Oct 07 08:26:28 Tien java[214549]: ... INFO ... spring-app : HikariPool-1 - Start completed.
Oct 07 08:26:28 Tien java[214549]: ... INFO ... spring-app : Database connected: MySQL 8.0.46 | jdbc:mysql://localhost:3306/springboot_db | user=spring-admin@localhost
Oct 07 08:26:28 Tien java[214549]: ... INFO ... spring-app : Tomcat started on port 8082 (http) with context path ''
Oct 07 08:26:28 Tien java[214549]: ... INFO ... spring-app : Started SpringAppApplication in 2.345 seconds
```

Trạng thái **`active (running)`**, `Loaded: ... enabled` xác nhận dịch vụ đã bật tự khởi động.

### 6.2. `ss -tlnp | grep 8082` → cổng do tiến trình Java lắng nghe

```console
$ ss -tlnp | grep 8082
LISTEN 0      50    *:8082    *:*    users:(("java",pid=214549,fd=7))
```

Cổng **8082** đang LISTEN bởi tiến trình **`java`** (PID 214549). `*:8082` là địa chỉ wildcard
dual-stack (IPv4 + IPv6) — đúng cách một ứng dụng Spring Boot/Tomcat bind mặc định; kết nối IPv4
tới `127.0.0.1:8082` vẫn thành công (kiểm chứng ở mục 6.4).

### 6.3. Xác nhận tiến trình chạy dưới `spring-runner` (non-root)

```console
$ MP=$(systemctl show -p MainPID --value spring-app.service)
$ ps -o user,uid,pid,ppid,cmd -p "$MP"
USER       UID     PID    PPID CMD
spring-+   997  214549       1 /usr/bin/java -jar /opt/spring-app/app.jar
```

→ Tiến trình chính chạy dưới **`spring-runner` (uid=997)**, **không phải `root`**.

### 6.4. Kiểm chứng ứng dụng hoạt động + kết nối DB

```console
$ curl -s -o /dev/null -w "GET /                 -> HTTP %{http_code}\n" http://127.0.0.1:8082/
GET /                 -> HTTP 200
$ curl -s -o /dev/null -w "GET /actuator/health  -> HTTP %{http_code}\n" http://127.0.0.1:8082/actuator/health
GET /actuator/health  -> HTTP 200
$ curl -s http://127.0.0.1:8082/actuator/health
{"status":"UP","components":{"db":{"status":"UP","details":{"database":"MySQL","validationQuery":"isValid()"}}}}
```

Ứng dụng ghi được vào DB qua JDBC (chứng minh user `spring-admin` thực sự kết nối `springboot_db`):

```console
$ mysql -h127.0.0.1 -uspring-admin -p'SpringSecure@123' springboot_db -e "SELECT COUNT(*) AS startup_rows FROM startup_log;"
startup_rows
3
```

### 6.5. Kiểm chứng tự động restart sau 10 giây khi crash (`Restart=on-failure`)

Mô phỏng sự cố sập đột ngột bằng `SIGKILL` tiến trình chính rồi đo thời điểm khởi động lại:

```console
$ OLD=$(systemctl show -p MainPID --value spring-app.service); echo "Before: MainPID=$OLD  $(date -u +%H:%M:%S)"
Before: MainPID=214549  08:26:48
$ kill -9 "$OLD"
$ # vòng lặp chờ PID mới
After : MainPID=214793  08:26:58  (restarted)
```

Đối chiếu `journalctl -u spring-app.service`:

| Thời điểm | Sự kiện |
|---|---|
| `08:26:48` | `Main process exited, code=killed, status=9/KILL` → `Failed with result 'signal'.` |
| `08:26:58` | `Scheduled restart job, restart counter is at 1.` → `Started spring-app.service ...` |

→ Đúng **10 giây** (`RestartSec=10`), dịch vụ **tự khởi động lại thành công** sau khi bị kill bất thường.

### 6.6. Tự khởi động cùng hệ thống

```console
$ systemctl is-enabled spring-app.service
enabled
```

→ Nhờ `WantedBy=multi-user.target`, dịch vụ sẽ tự chạy mỗi khi máy khởi động.

## 7. Giải thích cơ chế

- **Phân quyền DB theo nguyên tắc least privilege:** `spring-admin` chỉ có `ALL PRIVILEGES` trên
  `springboot_db.*`; ngoài database đó chỉ là `USAGE`. Nếu credential ứng dụng bị lộ, thiệt hại bị giới hạn
  trong một database thay vì toàn bộ server MySQL.
- **Vì sao dùng JDBC URL có `allowPublicKeyRetrieval=true&useSSL=false`:** MySQL 8 xác thực bằng
  `caching_sha2_password`; khi kết nối không TLS, driver cần được phép lấy public key của server để hoàn tất
  bắt tay (môi trường local/dev). Trên production nên bật TLS (`useSSL=true`) và quản lý key an toàn.
- **Non-root + `nologin`:** tiến trình chỉ có đặc quyền của `spring-runner`; shell `/usr/sbin/nologin` chặn
  đăng nhập trực tiếp → giảm thiệt hại nếu ứng dụng bị khai thác (defense in depth).
- **`Type=simple`:** phù hợp tiến trình foreground dài hạn (JVM). systemd coi PID của `ExecStart` là tiến trình
  chính → theo dõi/restart chính xác.
- **`Restart=on-failure` vs `always`:** `on-failure` chỉ restart khi thoát *bất thường* (exit ≠ 0 hoặc bị
  signal như `SIGKILL`). Khi ta chủ động `systemctl stop`, systemd biết đó là dừng theo yêu cầu nên **không**
  restart — đúng hành vi mong muốn. `RestartSec=10` tạo khoảng nghỉ tránh vòng lặp "crash → restart" liên tục.
- **`After`/`Wants` với `mysql.service`:** đảm bảo MySQL khởi động trước ứng dụng (đúng thứ tự phụ thuộc),
  trong khi `Wants` (không phải `Requires`) giữ dịch vụ không bị "cứng" nếu MySQL tạm thời vắng mặt.
- **`daemon-reload`:** bắt buộc mỗi khi tạo/sửa tệp `.service` để systemd nạp lại cấu hình unit.

## 8. Kết luận

- Đã tạo database **`springboot_db`** và user **`spring-admin`/`SpringSecure@123`** với **toàn quyền chỉ trên
  database này**; đăng nhập và ghi dữ liệu qua JDBC thành công.
- Đã tạo tài khoản hệ thống **`spring-runner`** (uid=997, `/usr/sbin/nologin`).
- Đã tự viết tệp unit **`/etc/systemd/system/spring-app.service`** chạy dưới user non-root, `ExecStart` đúng
  `/usr/bin/java -jar /opt/spring-app/app.jar`, `Restart=on-failure` + `RestartSec=10`, tự khởi động khi boot.
- Kết quả mong đợi: dịch vụ **`active (running)`** và **`enabled`**; cổng **8082** do tiến trình **`java`**
  chạy dưới **`spring-runner`** lắng nghe (`ss -tlnp | grep 8082`).
- Đã kiểm chứng thực tế: `kill -9` tiến trình → systemd **tự khởi động lại sau đúng 10 giây**.

## Phụ lục — Tóm tắt lệnh đã dùng

```bash
# 1. Khởi tạo database + user MySQL (least privilege)
sudo mysql < db/setup.sql

# 2. Tạo user hệ thống không shell
sudo useradd -r -s /usr/sbin/nologin spring-runner

# 3. Build & triển khai ứng dụng
cd app && ./build.sh          # -> /opt/spring-app/app.jar + application.properties + driver

# 4. Cài đặt & kích hoạt dịch vụ
sudo cp spring-app.service /etc/systemd/system/spring-app.service
sudo systemctl daemon-reload
sudo systemctl enable --now spring-app.service

# 5. Kiểm tra (theo đề bài)
sudo systemctl status spring-app.service
ss -tlnp | grep 8082
sudo journalctl -u spring-app.service --no-pager

# 6. Kiểm chứng tự restart (on-failure, sau 10 giây)
sudo kill -9 "$(systemctl show -p MainPID --value spring-app.service)"

# (tuỳ chọn) dừng / tắt dịch vụ
# sudo systemctl stop    spring-app.service
# sudo systemctl disable spring-app.service
```
