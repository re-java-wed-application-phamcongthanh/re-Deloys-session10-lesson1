# Bài 1: Báo Cáo Kỹ Thuật Phân Tích Kiến Trúc Container vs Virtual Machine

## 🎯 1. Mục Tiêu
- Phân tích chuyên sâu sự khác biệt kiến trúc giữa ảo hóa phần cứng (VM/Hypervisor) và ảo hóa hệ điều hành (Container Engine).
- Trình bày cơ chế cô lập kernel (Namespaces) và quản lý tài nguyên (Control Groups - cgroups) của Linux.
- Đánh giá hiện trạng hạ tầng Docker thông qua các thông số trích xuất từ lệnh `docker version` và `docker info`.

---

## 📁 2. Tài Liệu Nộp Bài
1. **Báo cáo kỹ thuật chi tiết**: [REPORT.md](file:///c:/hoccode/Devops%20Fundamentals/BaiTap/session10/bai1/REPORT.md)
   - Bảng so sánh 5 tiêu chí: Dung lượng Image, Thời gian khởi động, Hao phí OS, Mức độ cô lập, Khả năng mở rộng.
   - Giải thích chi tiết 6 Namespaces chính (PID, NET, MNT, IPC, UTS, USER) và cơ chế cgroups (CPU Quota, Memory Limit, BlkIO, PIDs Limit).
   - Phân tích các thông số vận hành thực tế trên hệ thống.
2. **File nhật ký log lệnh**: [docker_info.log](file:///c:/hoccode/Devops%20Fundamentals/BaiTap/session10/bai1/docker_info.log)

---

## 💻 3. Các Lệnh Kiểm Tra Hệ Thống
```bash
docker version
docker info
docker info | grep -i cgroup
```

**Thông số trích xuất:**
- **Server Version**: `29.7.2`
- **Cgroup Driver**: `cgroupfs` (Cgroup Version 2)
- **Storage Driver**: `overlayfs`
- **Số lượng Containers**: `13`
- **Số lượng Images**: `12`
