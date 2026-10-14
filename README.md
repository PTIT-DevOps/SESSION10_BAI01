# BT01 - Báo cáo kỹ thuật phân tích kiến trúc Container vs Virtual Machine

## 1. Bảng so sánh Virtual Machine và Container

Doanh nghiệp chuyển từ VPS (máy chủ ảo truyền thống) sang Docker vì VPS phải nhân bản toàn bộ
hệ điều hành cho mỗi ứng dụng, trong khi Docker chỉ đóng gói ứng dụng và các thư viện phụ thuộc
rồi dùng chung kernel của máy host.

| Tiêu chí | Virtual Machine | Container |
| --- | --- | --- |
| Dung lượng Image | 1 GB - 10 GB, bao gồm toàn bộ guest OS (kernel, init, dịch vụ hệ thống) | 5 MB - 500 MB, chỉ gồm ứng dụng và thư viện cần thiết |
| Thời gian khởi động | 30 giây - vài phút, phải boot guest OS và khởi động các dịch vụ | Dưới 1 giây - vài giây, chỉ khởi tạo process bằng runc |
| Hao phí OS | Mỗi VM có kernel riêng và chạy đầy đủ dịch vụ hệ thống, tốn CPU/RAM ngay cả khi ứng dụng nhàn rỗi | Dùng chung kernel host, không nhân bản OS, gần như không có hao phí nền |
| Mức độ cô lập | Rất cao, cô lập ở mức phần cứng ảo hóa, an toàn hơn khi chạy mã không tin cậy | Trung bình, cô lập ở mức process qua Namespaces và cgroups, dùng chung kernel |
| Khả năng mở rộng | Chậm, mỗi bản mở rộng tốn thêm tài nguyên OS, thời gian scale tính bằng phút | Nhanh, chỉ cần chạy thêm container, thời gian scale tính bằng giây |

## 2. Vai trò của Namespaces

Namespaces là tính năng của kernel Linux cho phép một process (hoặc nhóm process) có góc nhìn
riêng về tài nguyên hệ thống. Container Engine tạo một tập Namespaces mới cho mỗi container, nhờ
đó các container trên cùng một máy không nhìn thấy tài nguyên của nhau.

- PID namespace: mỗi container có cây tiến trình riêng, process đầu tiên trong container nhận PID 1
  và không nhìn thấy tiến trình của host hay của container khác.
- Network namespace: mỗi container có card mạng ảo, bảng định tuyến, iptables và cổng riêng. Đây là
  lý do hai container có thể cùng lắng nghe cổng 3000 mà không xung đột, và là nền tảng để Docker
  DNS phân giải tên container thành IP nội bộ.
- Mount namespace: mỗi container có cây thư mục riêng, chỉ thấy các lớp filesystem của image và các
  volume được gắn vào, không truy cập được filesystem của host.
- UTS namespace: cho phép container có hostname riêng, độc lập với hostname của máy host.
- IPC namespace: tách hàng đợi message và shared memory giữa các container.
- USER namespace: ánh xạ user bên trong container sang user khác trên host, giới hạn quyền khi bị
  thoát khỏi container.

## 3. Vai trò của Control Groups (cgroups)

Namespaces giới hạn những gì container nhìn thấy, còn Control Groups giới hạn những gì container
được phép dùng. cgroups phiên bản 2 tổ chức các process thành một cây thống nhất và cung cấp các
bộ điều khiển tài nguyên:

- CPU: `cpu.max` khống chế tổng thời gian CPU (ví dụ 0.5 CPU), `cpu.weight` phân chia mức ưu tiên
  khi các container tranh chấp CPU.
- RAM: `memory.max` là ngưỡng bộ nhớ tối đa của container, `memory.swap.max` giới hạn swap. Khi
  container vượt ngưỡng, kernel kích hoạt OOM killer và chỉ kết thúc process trong container đó,
  các container khác trên máy vẫn chạy bình thường.
- Ngoài ra còn có `pids.max` giới hạn số tiến trình và bộ điều khiển cho I/O đĩa (io.max).

Các giới hạn này được thiết lập qua tham số `--cpus`, `--memory`, `--pids-limit` của `docker run`
hoặc qua `deploy.resources` trong Docker Compose.

## 4. Trích xuất thông số cơ bản từ docker info

```bash
devops@0310263186:~$ docker version
Client: Docker Engine - Community
 Version:           27.3.1
 API version:       1.46
 Go version:        go1.22.7
 Git commit:        ce12230
 Built:             Fri Sep 20 11:40:59 2024
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          27.3.1
  API version:      1.46 (minimum version 1.24)
  Go version:       go1.22.7
  Git commit:       41ca978
  Built:            Fri Sep 20 11:40:59 2024
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.7.22
  GitCommit:        7f7fdf5fed64eb6a7caf99b3e12efcf9d60e311c
 runc:
  Version:          1.1.14
  GitCommit:        v1.1.14-0-g2c9f560
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

```bash
devops@0310263186:~$ docker info | grep -i cgroup
 Cgroup Driver: systemd
 Cgroup Version: 2
```

```bash
devops@0310263186:~$ docker info
Server:
 Containers: 3
  Running: 1
  Paused: 0
  Stopped: 2
 Images: 6
 Server Version: 27.3.1
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Using metacopy: false
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Kernel Version: 5.15.0-122-generic
 Operating System: Ubuntu 22.04.5 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 1.92GiB
 Name: 0310263186
 Docker Root Dir: /var/lib/docker
```

| Thông số | Giá trị |
| --- | --- |
| Server Version | 27.3.1 |
| Cgroup Driver | systemd |
| Cgroup Version | 2 |
| Containers | 3 (1 Running, 2 Stopped) |
| Images | 6 |

Cả `docker version` và `docker info` đều thực thi thành công bằng user `devops` không cần `sudo`,
nghĩa là user này đã thuộc nhóm `docker`.
