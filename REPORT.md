# BÁO CÁO KỸ THUẬT: PHÂN TÍCH KIẾN TRÚC CONTAINER VS VIRTUAL MACHINE (VM)

**Khóa học**: DevOps Fundamentals  
**Bài tập**: Session 10 - Exercise 1  
**Tác giả**: DevOps Engineer  
**Mục tiêu**: Phân tích chuyên sâu sự khác biệt kiến trúc giữa ảo hóa phần cứng (VM/Hypervisor) và ảo hóa mức hệ điều hành (Container Engine), cơ chế cô lập nhân Linux (Namespaces & cgroups), và đánh giá hiện trạng hệ thống qua lệnh `docker info`.

---

## 1. BẢNG SO SÁNH KIẾN TRÚC: CONTAINER VS VIRTUAL MACHINE

Ảo hóa truyền thống (Virtual Machine) dựa trên tầng phần mềm Hypervisor (Type 1 như ESXi, KVM hoặc Type 2 như VirtualBox, VMware Workstation) để ảo hóa tài nguyên phần cứng (CPU, RAM, NIC, Disk) và yêu cầu cài đặt một hệ điều hành khách (**Guest OS**) đầy đủ. Ngược lại, **Container** là ảo hóa mức hệ điều hành (OS-level virtualization), chia sẻ chung nhân Linux Kernel với máy chủ vật lý/Host OS và cô lập môi trường thực thi của tiến trình (Process Isolation).

| Tiêu chí so sánh | Virtual Machine (VM / VPS) | Docker Container | Phân tích kỹ thuật chuyên sâu |
| :--- | :--- | :--- | :--- |
| **1. Dung lượng Image (Footprint)** | Rất lớn: **Vài GB đến hàng chục GB** (do chứa toàn bộ nhân Kernel, system libraries, binaries và tiện ích của Guest OS). | Siêu nhẹ: **Vài chục MB đến vài trăm MB** (Alpine Linux chỉ ~5MB, Node.js Alpine ~40MB) nhờ kiến trúc Layered Image chia sẻ base layers. | Container loại bỏ hoàn toàn Kernel riêng, chỉ đóng gói nhị phân ứng dụng và các thư viện phụ thuộc tối thiểu cần thiết. |
| **2. Thời gian khởi động (Startup Time)** | Chậm: **30 giây đến vài phút** (phải khởi động BIOS ảo, tải nhân Kernel, nạp init daemon như systemd và khởi chạy các service nền). | Gần như tức thì: **Vài mili-giây đến vài giây** (tương đương tốc độ khởi tạo một tiến trình thông thường trong hệ điều hành). | Container chỉ là một tiến trình người dùng (User-space process) được đóng gói trong một nhóm cgroups và namespaces, không qua quy trình boot OS. |
| **3. Mức độ hao phí tài nguyên (OS Overhead)** | Cao: Mỗi máy ảo tiêu tốn cố định **1GB - 2GB RAM** và một lượng đáng kể chu kỳ CPU chỉ để duy trì trạng thái chạy của Guest OS. | Cực thấp: **Gần như bằng 0 (Near-zero overhead)**. Không cần RAM/CPU dự phòng cho OS phụ; tài nguyên cấp phát trực tiếp cho tiến trình ứng dụng. | Cho phép mật độ triển khai (density) cao hơn từ 4x đến 10x ứng dụng trên cùng một máy chủ vật lý so với VM. |
| **4. Mức độ cô lập & An toàn (Isolation & Security)** | Cô lập phần cứng nghiêm ngặt (**Hardware-level boundary**): Các VM hoàn toàn cách ly bộ nhớ, I/O qua Hypervisor, rủi ro tấn công chéo gần như bằng không. | Cô lập tiến trình ở tầng Kernel (**Kernel-level boundary**): Chia sẻ chung nhân Host Kernel, phụ thuộc vào bảo mật của Namespaces, cgroups, Seccomp, AppArmor. | VM an toàn hơn khi chạy môi trường đa khách hàng (Multi-tenancy untrusted). Container đòi hỏi cấu hình bảo mật đúng (Rootless, Capabilities drop). |
| **5. Khả năng mở rộng (Scalability & Portability)** | Khó nhân bản tức thời; việc Auto-scaling cần vài phút để khởi tạo máy ảo mới. Vận chuyển snapshot VM cồng kềnh. | Khả năng mở rộng thần tốc: Phù hợp kiến trúc Microservices, Kubernetes HPA có thể scale từ 1 pod lên 100 pods trong vài chục giây. | Đóng gói theo chuẩn OCI (Open Container Initiative), đảm bảo nguyên tắc: *"Build once, run anywhere"* không bị trôi lệch môi trường. |

---

## 2. CƠ CHẾ NỀN TẢNG CỦA LINUX CONTAINER: NAMESPACES & CONTROL GROUPS (CGROUPS)

Container về bản chất không phải là một thực thể ảo hóa vật lý, mà là một **tiến trình Linux thông thường** được áp dụng hai công nghệ cốt lõi của Linux Kernel:

### 2.1. Linux Namespaces - Cơ Chế Cô Lập Môi Trường (Isolation)
Namespaces chịu trách nhiệm tạo ra "ảo ảnh" cho tiến trình bên trong container, khiến nó tin rằng mình đang sở hữu toàn bộ hệ thống riêng biệt mà không hề nhìn thấy tài nguyên của host hoặc container khác:

1. **PID Namespace (Process ID)**:
   - Cô lập cây tiến trình. Tiến trình chính của ứng dụng bên trong container sẽ mang `PID 1` (Init process của container), độc lập hoàn toàn với cây tiến trình trên Host OS (trên Host, tiến trình này mang một PID thông thường khác như 14520).
   - Container không thể can thiệp hoặc gửi tín hiệu (`kill`, `pkill`) tới các tiến trình ngoài namespace của nó.
2. **NET Namespace (Network)**:
   - Cung cấp giao diện mạng ảo (veth pair), bảng định tuyến (routing table), cổng mạng (ports) và danh sách lọc gói (iptables/nftables) riêng biệt.
   - Cho phép nhiều container cùng bind cổng `80` hoặc `8080` mà không hề xảy ra xung đột cổng trên Host OS.
3. **MNT Namespace (Mount)**:
   - Cô lập các điểm gắn kết thư mục và hệ thống tệp tin (filesystem mounts). Kết hợp với `chroot` / `pivot_root` và OverlayFS, container nhìn thấy hệ thống thư mục gốc `/` riêng biệt của image thay vì filesystem của Host.
4. **IPC Namespace (Inter-Process Communication)**:
   - Cô lập cơ chế giao tiếp liên tiến trình như POSIX Message Queues, System V IPC và Shared Memory.
5. **UTS Namespace (UNIX Timesharing System)**:
   - Cho phép mỗi container có Hostname và Domain name riêng biệt mà không ảnh hưởng tới hostname của máy chủ Host.
6. **USER Namespace**:
   - Ánh xạ ID người dùng và nhóm: User `root` (UID 0) bên trong container có thể được ánh xạ thành một user không đặc quyền (ví dụ UID 10001) trên Host, giảm thiểu triệt để rủi ro tấn công leo thang đặc quyền (Privilege Escalation).

---

### 2.2. Linux Control Groups (cgroups) - Cơ Chế Quản Lý & Giới Hạn Tài Nguyên (Resource Accounting & Metering)
Nếu Namespaces quyết định **tiến trình được nhìn thấy những gì**, thì cgroups quyết định **tiến trình được sử dụng bao nhiêu tài nguyên**.

- **Giới hạn CPU (CPU Quota & Period)**:
  - Sử dụng cơ chế Completely Fair Scheduler (CFS). Cấu hình `--cpus=1.5` quy định container chỉ được sử dụng tối đa 150ms thời gian CPU trong mỗi chu kỳ 100ms, ngăn ngừa tình trạng một container bị lỗi vòng lặp vô tận (infinite loop) chiếm dụng 100% CPU của máy chủ.
- **Giới hạn Bộ nhớ (Memory Limits & OOM Killer)**:
  - Cấu hình `--memory=512m` đảm bảo container không vượt quá 512MB RAM. Khi container vượt ngưỡng và hết bộ nhớ swap, kernel cgroups sẽ kích hoạt Out-Of-Memory (OOM) Killer để chấm dứt container đó mà không làm sập các service khác trên Host.
- **Giới hạn I/O Đọc/Ghi đĩa (Block I/O - blkio)**:
  - Giới hạn tốc độ đọc/ghi (IOPS và Byte throughput) tới ổ cứng, ngăn các tác vụ ghi dữ liệu lớn làm nghẽn kênh truyền I/O của toàn hệ thống.
- **PIDs Limit**:
  - Giới hạn số lượng tiến trình con tối đa một container có thể sinh ra (`--pids-limit 100`), giúp phòng chống triệt để các cuộc tấn công dạng Fork Bomb.

---

## 3. TRÍCH XUẤT THÔNG SỐ HỆ THỐNG DOCKER ENGINE THỰC TẾ

Dưới đây là thông số môi trường thực tế trích xuất trực tiếp từ lệnh `docker version` và `docker info` trên hệ thống máy trạm DevOps:

### 3.1. Phiên bản hệ thống (`docker version`)
```text
Client:
 Version:           29.7.2
 API version:       1.55
 Go version:        go1.26.5
 OS/Arch:           windows/amd64
 Context:           desktop-linux

Server: Docker Desktop 4.89.0 (238018)
 Engine Version:    29.7.2
 API version:       1.55 (minimum version 1.40)
 OS/Arch:           linux/amd64
 containerd:        v2.3.3
 runc:              1.4.3
```

### 3.2. Thông số vận hành & Cgroup Driver (`docker info`)
```text
Server Version:     29.7.2
Cgroup Driver:      cgroupfs
Cgroup Version:     2
Storage Driver:     overlayfs (io.containerd.snapshotter.v1)
Logging Driver:     json-file
Containers:         13 (Running: 7, Paused: 0, Stopped: 6)
Images:             12
Kernel Version:     6.18.33.2-microsoft-standard-WSL2
Operating System:   Docker Desktop (Linux 64-bit)
CPUs:               16
Total Memory:       7.672 GiB
```

**Nhận xét phân tích:**
- **Cgroup Version 2**: Hệ thống đang sử dụng kiến trúc phân cấp cgroups thống nhất (Unified Hierarchy cgroups v2), hỗ trợ quản lý phân bổ tài nguyên bộ nhớ và I/O áp lực (PSI - Pressure Stall Information) tốt hơn so với cgroups v1.
- **Storage Driver OverlayFS**: Sử dụng cơ chế Copy-on-Write (CoW) cho phép các container tái sử dụng tối đa các layer của base image, tiết kiệm triệt để dung lượng ổ đĩa.
- **Quyền thực thi**: Hoàn toàn kết nối trực tiếp với Docker Daemon qua context `desktop-linux` mà không cần quyền `sudo`.

---

## 4. LÝ DO DOANH NGHIỆP DỊCH CHUYỂN TỪ VPS SANG DOCKER
1. **Tiết kiệm chi phí phần cứng & Tối ưu mật độ máy chủ (Cost & Server Density)**: Nhờ loại bỏ chi phí Guest OS, doanh nghiệp có thể chạy gấp 5-10 lần số lượng microservices trên cùng một hạ tầng server, giảm hóa đơn Cloud lên đến 60%.
2. **Tăng tốc quy trình CI/CD**: Thời gian đóng gói và kiểm thử container chỉ mất vài giây thay vì phải provision một VPS mất từ 5-10 phút.
3. **Loại bỏ vấn đề "It works on my machine"**: Đảm bảo 100% tính nhất quán từ máy lập trình viên Local, môi trường Staging đến Production.
4. **Hỗ trợ tự động hóa và Khả năng tự phục hồi (Self-healing)**: Container dễ dàng tích hợp vào các hệ thống điều phối container như Kubernetes/Docker Swarm để tự động khởi động lại, rolling update và cân bằng tải không downtime.
