# UISPTITv2 — Quản lý đăng ký tín chỉ đa cơ sở

> Đồ án cuối kỳ môn **Cơ sở dữ liệu phân tán (CSDLPT)**
> Hệ thống đăng ký tín chỉ cho một trường đại học có nhiều cơ sở đào tạo, xây trên **SQL Server** với phân mảnh ngang, phân mảnh dẫn xuất, nhân bản một chiều và truy vấn phân tán.

---

## Bài toán

Một trường có nhiều cơ sở đào tạo đặt tại các tỉnh/thành khác nhau. Mỗi cơ sở vận hành gần như độc lập hằng ngày — tuyển sinh, mở lớp, giảng dạy, nhập điểm — nhưng vẫn dùng chung một chương trình đào tạo và cần thống kê toàn hệ thống.

**Bốn lý do bắt buộc phải phân tán:**

1. **Địa lý** — các cơ sở cách nhau ~1.700 km. Ràng buộc vật lý, không tối ưu bằng phần cứng được
2. **Tự chủ quản trị** — ranh giới quyền quản trị dữ liệu phải trùng ranh giới tổ chức
3. **Cô lập lỗi** — CSDL tập trung nghĩa là một sự cố ở trung tâm làm toàn bộ mọi cơ sở ngừng hoạt động
4. **Yêu cầu môn học**

---

## Kiến trúc

```
              ┌──────────────────────────────────────────┐
              │  UIS_MASTER — VAI TRÒ MASTER             │
              │  Publisher · Distributor                 │
              │  CoSo · Khoa · CTDT · MonHoc · HocKy     │
              │  DanhBaSinhVien                          │
              └────┬──────────────┬──────────────┬───────┘
                   │              │              │
          Transactional Replication một chiều (~vài giây)
                   ▼              ▼              ▼
      ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
      │    UIS_HCM     │ │    UIS_HN      │ │    UIS_DN      │
      │  Subscriber    │ │  Subscriber    │ │  Subscriber    │
      │  + mảnh vận    │ │  + mảnh vận    │ │  + mảnh vận    │
      │    hành riêng  │ │    hành riêng  │ │    hành riêng  │
      └────────────────┘ └────────────────┘ └────────────────┘
              └──── Linked Server hình sao ────┘
                    (CHỈ cho thống kê toàn hệ thống)
```

**Hai chế độ ghi** — đây là điểm cốt lõi của thiết kế:

| Loại dữ liệu | Chế độ ghi |
|---|---|
| **Vận hành** — sinh viên, lớp học phần, đăng ký, điểm | **Mỗi cơ sở là master của chính mình.** Ghi độc lập hoàn toàn. ~99,9% lượt ghi |
| **Tham chiếu** — danh mục môn học, khoa, học kỳ, danh bạ | **Một nơi ghi, nhân bản ra mọi nơi.** ~15 lượt ghi/ngày |

Hệ thống **không** phải single-master. Ghi được phân hoạch theo mảnh cho dữ liệu nghiệp vụ, và single-master cho dữ liệu tham chiếu.

---

## Kỹ thuật phân tán áp dụng

| Kỹ thuật | Áp dụng cho |
|---|---|
| **Phân mảnh ngang** | `SinhVien`, `GiangVien`, `TaiKhoan`, `DotDangKy`, `LopHocPhan` |
| **Phân mảnh ngang dẫn xuất** | `DangKyHocPhan` (bậc 1, `⋉ LopHocPhan`) · `Diem` (bậc 2) |
| **Nhân bản một chiều** | Danh mục dùng chung + **danh bạ định vị** `DanhBaSinhVien` |
| **Linked Server** | Chỉ cho thống kê toàn hệ thống — `OPENQUERY` / `EXEC … AT` |
| **Tối ưu truy vấn phân tán** | Aggregate pushdown / semi-join, có benchmark đo bằng số liệu |
| **Xử lý tương tranh** | `UPDATE … WHERE SoLuongDaDangKy < SoLuongToiDa` + 4 lớp ràng buộc |
| **Saga + Outbox** | Đăng ký liên cơ sở, thay cho distributed transaction / 2PC |
| **Ba mức trong suốt** | Fragmentation · Location · Replication transparency |

---

## Điểm đặc biệt: X-Ray Phân Tán

Môn CSDLPT dạy toàn những thứ **không nhìn thấy được**. X-Ray làm chúng hiện ra theo thời gian thực — mọi response mang theo một trace, giao diện vẽ lại đúng đường đi của request đó qua hệ thống:

```
┌─ X-RAY ──────────────────────────────────────────────┐
│  GET /api/thong-ke/mon-hoc?ma=INT1339                 │
│                                                        │
│      [HCM] ●━━━━━━━  cục bộ      ·  1 dòng ·   9ms    │
│      [HN ] ●━━━━━━━  OPENQUERY   ·  1 dòng ·  87ms    │
│      [ĐN ] ●━━━━━━━  OPENQUERY   ·  1 dòng ·  91ms    │
│                                                        │
│  Dòng qua mạng: 2   (đã GROUP BY tại từng site)       │
│  Chiến lược: OPENQUERY      [ đổi sang four-part ]    │
└────────────────────────────────────────────────────────┘
```

Bấm *"đổi sang four-part"* — cùng câu hỏi, hiện ngay **84.213 dòng qua mạng · 3.412ms**. Benchmark không còn là bảng tĩnh trong báo cáo mà là thứ chạy được trực tiếp.

---

## Cấu trúc repo

```
uisptitv2/
├── docs/
│   ├── UISPTITv2-Thiet-Ke-v2.md   ← TÀI LIỆU DUY NHẤT của dự án
│   ├── bao-cao/                    ← bản Word nộp thầy
│   ├── diagrams/                   ← ERD, lược đồ phân mảnh/ánh xạ/định vị
│   └── screenshots/                ← ảnh cài đặt từng bước (chụp từ tuần 1)
├── db/
│   ├── 00-create-databases.sql     ← UIS_MASTER · UIS_HCM · UIS_HN · UIS_DN
│   ├── 01-schema-master.sql        02-schema-site.sql
│   ├── 03-roles-grants.sql         04-triggers.sql
│   ├── 05-linked-server.sql        06-replication/
│   └── 99-seed/
├── apps/
│   ├── api/                        ← Spring Boot (modular monolith)
│   └── web/                        ← React + Vite
└── bench/                          ← sinh tải + kịch bản benchmark
```

---

## Bắt đầu từ đâu

Toàn bộ thiết kế nằm trong **một tài liệu duy nhất**: [`docs/UISPTITv2-Thiet-Ke-v2.md`](docs/UISPTITv2-Thiet-Ke-v2.md)

| Bạn đang cần… | Đọc mục |
|---|---|
| Nắm nhanh dự án trong 5 phút | **0.1** bảng quyết định · **0.2** X-Ray · **C0** hai chế độ ghi |
| Viết báo cáo mục 2.1 / 2.2.1 / 2.2.2 | **Phần A** / **B** / **C** |
| Làm cài đặt vật lý (mục 3 đề bài) | **Phần F** chi tiết · **I5** checklist tick nhanh |
| Tra nhanh một bảng | **I6** danh sách bảng |
| Biết còn gì chưa chốt | **I4** việc còn treo |
| Lo về máy móc, chi phí, uptime | **I2b** vận hành máy chủ |

---

## Yêu cầu môi trường

| Hạng mục | Yêu cầu |
|---|---|
| CSDL | **SQL Server Developer Edition** — bắt buộc. Express không làm được Publisher và không có SQL Server Agent |
| Máy | 2–4 máy Windows 10/11 · 8GB RAM · 20GB đĩa trống |
| Mạng | Radmin VPN (hoặc Tailscale) |
| Backend | JDK 17+ · Maven |
| Frontend | Node 20+ |
| **Chi phí hạ tầng** | **0đ** — chạy hoàn toàn trên máy của nhóm |

> Hệ thống **không cần chạy 24/7**. Với retention đã cấu hình (subscription expiration 30 ngày), một máy có thể tắt tới 30 ngày mà vẫn bắt kịp. Chi tiết ở mục **I2b**.

---

## Trạng thái

**Giai đoạn hiện tại:** thiết kế đã chốt, chưa bắt đầu cài đặt.

Nguyên tắc thi công: **không viết dòng code ứng dụng nào trước khi phần cài đặt vật lý đã PASS và đã chụp đủ screenshot.**

Việc còn treo — xem mục **I4**:

- [ ] File Excel phân công đề tài của giảng viên
- [ ] Tài liệu hướng dẫn Replication của giảng viên
- [ ] Số liệu quy mô thật (thay giả định trong bảng tần suất)
- [ ] Chốt số cơ sở và kiểu instance — cuối tuần 1, sau spike replication
- [ ] Chốt người giữ máy chủ từng site + lịch buổi làm việc cố định

---

## Lộ trình

| Tuần | Trọng tâm | |
|---|---|---|
| 1 | Phân tích · ERD · bảng tần suất · **spike replication (cổng chặn)** | Bắt buộc |
| 2 | Thiết kế phân mảnh/ánh xạ/định vị · schema · seed dữ liệu | Bắt buộc |
| 3 | **Cài đặt vật lý** — VPN · Linked Server · Publication | Bắt buộc |
| 4 | Trigger · phân quyền · transaction · test tương tranh | Bắt buộc |
| 5 | Ứng dụng nền tảng | Mở rộng |
| 6 | Đăng ký liên cơ sở — saga · outbox · projection | Mở rộng |
| 7 | X-Ray · benchmark · kịch bản sự cố | Mở rộng |
| 8 | Báo cáo · slide · demo · tuần đệm | Bắt buộc |

⚠️ **Cổng chặn cuối tuần 4:** phần bắt buộc phải xong trước khi đụng vào phần mở rộng.
