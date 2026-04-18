# 🤖 HỆ THỐNG ĐIỀU KHIỂN & GIÁM SÁT ROBOT SONG SONG 3RRR

> **Ứng dụng Mạng Nơ-ron Nhân tạo RBFNN và Thiết kế, Thi công Hệ thống Điều khiển Giám sát cho Robot Song song Phẳng 3 Bậc Tự do**

---

## 📋 Tổng quan Dự án

Đây là ứng dụng HMI (Human-Machine Interface) SCADA được xây dựng bằng **WPF C# (.NET 8)** nhằm điều khiển và giám sát thời gian thực Robot Song song Phẳng 3RRR 3 bậc tự do thông qua chuẩn giao tiếp **UART**. Bộ điều khiển phần cứng là vi điều khiển **STM32**.

Phần mềm không chỉ nhận và hiển thị dữ liệu từ STM32, mà còn **tính toán Động học thuận (FK)** và **Động học nghịch (IK)** trực tiếp trên C# để xác định **giá trị thực tế** và **sai số** của robot — cung cấp khả năng giám sát sâu mà không cần STM32 phải gửi thêm dữ liệu.

### 📌 Thông tin Đồ án
| Thông tin | Chi tiết |
|---|---|
| **Trường** | Đại học Bách khoa – Đại học Đà Nẵng |
| **Khoa** | Điện |
| **GVHD** | PGS. TS Lê Tiến Dũng |
| **SVTH** | Phan Hữu Anh Đức (MSSV: 105210311) |
| | Trương Bảo Quang (MSSV: 105210332) |
| **Lớp** | 21TDH2 |

---

## ✨ Tính năng nổi bật

| Tính năng | Mô tả |
|-----------|-------|
| **Digital Twin 2D** | Mô phỏng robot trực quan trên Canvas với cánh tay chủ động/bị động, mâm tam giác, lưới tọa độ, và đường vẽ quỹ đạo |
| **Tính toán Động học** | Tích hợp FK (Newton-Raphson) và IK (giải tích đóng) dịch từ MATLAB, so sánh giá trị mong muốn vs. thực tế |
| **4 Đồ thị thời gian thực** | Quỹ đạo XY, Theta, Momen, Sai số — 30 FPS, chọn kênh linh hoạt |
| **Edge Detection** | Tự động phát hiện sườn lên/xuống SystemState để ghi dữ liệu chính xác |
| **Lưu trữ SQLite** | Bulk Insert tối ưu, lưu 24 cột dữ liệu mỗi frame, hỗ trợ tra cứu + xuất CSV + xóa |
| **9 quỹ đạo** | Hình Vuông, Tròn, Đường Thẳng, Sao, Hạc, Chữ A/B/DUT/TDH |
| **Giám sát kết nối** | LED nhấp nháy cảnh báo mất UART > 500ms |

---

## 🖥️ Các màn hình chính

| Màn hình | Mô tả |
|---|---|
| `Login.xaml` | Màn hình đăng nhập bảo mật (Username: `admin`, Password: `123`) |
| `MainWindow.xaml` | **Trang chủ SCADA** – Digital Twin 2D + Điều khiển (Start/Stop/Home) + LED trạng thái |
| `Trends.xaml` | **Đồ thị thời gian thực** – Quỹ đạo XY (mong muốn + thực tế), Theta, Momen, Sai số |
| `Parameters.xaml` | **Trang thông số** – Hiển thị 24+ tham số kỹ thuật: mong muốn, thực tế, sai số |
| `History.xaml` | **Lịch sử hoạt động** – Nhật ký vận hành, bộ lọc, xuất CSV, xóa dữ liệu |

---

## 🚀 Cài đặt và Khởi chạy

### Yêu cầu hệ thống
- **OS:** Windows 10/11 (64-bit)
- **Runtime:** .NET 8.0 hoặc cao hơn
- **IDE:** Visual Studio 2022 (khuyến nghị)
- **Phần cứng:** STM32 Controller kết nối qua cáp USB-UART (cổng COM)

### Các bước khởi chạy
1. Clone hoặc mở Solution file `HMI_WPF.slnx` bằng Visual Studio 2022.
2. Restore NuGet packages (tự động khi mở Solution).
3. Nhấn `F5` hoặc `Ctrl+F5` để Build và chạy ứng dụng.
4. Đăng nhập với tài khoản:
   - **Tài khoản:** `admin`
   - **Mật khẩu:** `123`
5. Ứng dụng sẽ **tự động kết nối** cổng `COM3` với baudrate `115200`. Nếu cần đổi cổng, nhấn **Ngắt** → chọn cổng mới → nhấn **Kết nối**.
6. Chọn quỹ đạo → Nhấn **START** để bắt đầu vận hành.

---

## 🔗 Giao thức Truyền thông UART

### Cấu trúc Khung dữ liệu nhận (77 byte)

Dữ liệu từ STM32 gửi lên theo định dạng khung nhị phân cố định (~30Hz):

```
[Header: 0x7E 0x7E] [72 byte dữ liệu: 9 × double] [SystemState: 1 byte] [Terminator: 0x03 0x03]
```

| Offset | Kích thước | Biến | Ý nghĩa | Đơn vị |
|--------|-----------|------|---------|--------|
| 0–1 | 2 byte | Header | `0x7E 0x7E` — Đánh dấu đầu khung | — |
| 2–9 | 8 byte (double) | Theta1 | Góc mong muốn khớp 1 | rad |
| 10–17 | 8 byte (double) | Theta2 | Góc mong muốn khớp 2 | rad |
| 18–25 | 8 byte (double) | Theta3 | Góc mong muốn khớp 3 | rad |
| 26–33 | 8 byte (double) | PosX | Tọa độ X mong muốn | m |
| 34–41 | 8 byte (double) | PosY | Tọa độ Y mong muốn | m |
| 42–49 | 8 byte (double) | Alpha | Góc nghiêng mong muốn | rad |
| 50–57 | 8 byte (double) | Torque1 | Momen xoắn động cơ 1 | N·m |
| 58–65 | 8 byte (double) | Torque2 | Momen xoắn động cơ 2 | N·m |
| 66–73 | 8 byte (double) | Torque3 | Momen xoắn động cơ 3 | N·m |
| 74 | 1 byte | SystemState | 1 = Đang chạy, 0 = Đang dừng | — |
| 75–76 | 2 byte | Terminator | `0x03 0x03` — Đánh dấu cuối khung | — |

> **Lưu ý:** Giá trị double theo định dạng **Little Endian** (IEEE 754). Sai số (Error) và giá trị thực tế (Actual) **không có trong UART** — được tính toán trên C# qua module `RobotKinematics`.

### Cấu trúc Khung lệnh gửi xuống (8 byte)

| Byte | Giá trị | Ý nghĩa |
|------|---------|---------|
| 0–1 | `0x7E 0x7E` | Header |
| 2 | `1–9` | TrajectoryID (ID quỹ đạo) |
| 3 | `0 / 1` | Start flag |
| 4 | `0 / 1` | Stop flag |
| 5 | `0 / 1` | Home flag |
| 6–7 | `0x03 0x03` | Terminator |

### Bảng quỹ đạo

| ID | Tên quỹ đạo |
|----|-------------|
| 1 | Hình Vuông |
| 2 | Hình Tròn |
| 3 | Đường Thẳng |
| 4 | Hình Sao |
| 5 | Hình Hạc |
| 6 | Vẽ Chữ A |
| 7 | Vẽ Chữ B |
| 8 | Vẽ Chữ DUT |
| 9 | Vẽ Chữ TDH |

---

## 🔬 Tính toán Động học trên C#

Phần mềm tích hợp module `RobotKinematics.cs` thực hiện tính toán **hoàn toàn trên máy tính**, dịch trực tiếp từ mã MATLAB của đề tài:

| Phép tính | Đầu vào (từ UART) | Đầu ra (tính trên C#) | Thuật toán |
|-----------|-------------------|----------------------|-----------|
| **Động học thuận (FK)** | Theta1/2/3 (mong muốn) | ActualX, ActualY, ActualAlpha (thực tế) | Newton-Raphson (tối đa 300 vòng, ε = 1e-12) |
| **Động học nghịch (IK)** | PosX, PosY, Alpha (mong muốn) | ActualTheta1/2/3 (thực tế) | Giải tích đóng (closed-form) |
| **Sai số** | Mong muốn − Thực tế | ErrorX/Y/Alpha, ErrorTheta1/2/3 | Phép trừ |

### Thông số hình học Robot

| Ký hiệu | Giá trị | Ý nghĩa |
|---------|---------|---------|
| L1 | 0.20 m | Chiều dài cánh tay chủ động |
| L2 | 0.20 m | Chiều dài cánh tay bị động |
| L3 | 0.125/√3 ≈ 0.0722 m | Bán kính nội tiếp mâm di động |
| O1 | (0, 0) | Gốc động cơ 1 |
| O2 | (0.5, 0) | Gốc động cơ 2 |
| O3 | (0.25, 0.433) | Gốc động cơ 3 |

---

## 📦 Cấu trúc Thư mục

```
HMI_WPF/
├── HMI_WPF.slnx                  # Solution file
├── README.md                      # File này
├── docs/
│   ├── KIEN_TRUC_DU_AN.md        # Kiến trúc phần mềm chi tiết (tiếng Việt)
│   └── ARCHITECTURE.md           # Kiến trúc kỹ thuật
│
└── HMI_WPF/                      # Project chính
    ├── App.xaml / App.xaml.cs      # Điểm khởi động + SharedRobotViewModel + DB init
    ├── Login.xaml / .cs            # Màn hình đăng nhập
    ├── MainWindow.xaml / .cs       # Trang chủ SCADA (Digital Twin + Điều khiển)
    ├── Trends.xaml / .cs           # 4 biểu đồ thời gian thực (ScottPlot 5)
    ├── Parameters.xaml / .cs       # Bảng thông số kỹ thuật
    ├── History.xaml / .cs          # Lịch sử vận hành + Xuất CSV + Xóa
    │
    ├── Robot/                      # Giao tiếp phần cứng + Động học
    │   ├── RobotData.cs            # Mô hình dữ liệu (24+ thuộc tính)
    │   ├── UartManager.cs          # Singleton thu phát UART + tính FK/IK
    │   └── RobotKinematics.cs      # Module Động học thuận/nghịch 3RRR
    │
    ├── ViewModels/                 # Lớp xử lý logic (MVVM)
    │   ├── ViewModelBase.cs        # Lớp cơ sở (INotifyPropertyChanged)
    │   ├── RelayCommand.cs         # ICommand wrapper cho nút bấm
    │   └── RobotViewModel.cs       # ViewModel chính: Edge Detection + DB logging
    │
    ├── Models/                     # Cấu trúc CSDL
    │   └── DbModels.cs            # Bảng RunHistory (5 cột) + RunData (24 cột)
    │
    └── Services/                   # Dịch vụ
        └── DatabaseManager.cs      # Singleton SQLite: CRUD + Bulk Insert + Migration
```

---

## 🛠️ Công nghệ sử dụng

| Công nghệ | Phiên bản | Mục đích |
|---|---|---|
| C# / .NET | 8.0 | Ngôn ngữ & Runtime chính |
| WPF | .NET 8 | Framework giao diện |
| ScottPlot | 5.1.58 | Vẽ đồ thị thời gian thực (Scatter rebuild 30fps) |
| Microsoft.Data.Sqlite | 8.0.0 | Cơ sở dữ liệu SQLite (lưu lịch sử vận hành) |
| System.IO.Ports | 10.0.5 | Giao tiếp cổng COM / UART |
| MVVM Pattern | — | Tách biệt UI và logic xử lý |
| Singleton Pattern | — | UartManager, DatabaseManager, SharedRobotViewModel |
| Edge Detection | — | Phát hiện sườn trạng thái để kích hoạt ghi DB |

---

## 🔑 Các Design Patterns chính

| Pattern | Áp dụng ở | Mục đích |
|---------|----------|---------|
| **Singleton** | UartManager, DatabaseManager, SharedRobotViewModel | Đảm bảo 1 instance duy nhất, chia sẻ giữa mọi cửa sổ |
| **MVVM** | ViewModelBase + RelayCommand + RobotViewModel | Tách UI (XAML) khỏi logic, hỗ trợ Data Binding |
| **Observer/Event** | DataReceived, TrajectoryStarted, RunStarted | Broadcast dữ liệu đến tất cả trang đang mở |
| **Edge Detection** | RobotViewModel.UpdateData() | Phát hiện sườn 0→1 / 1→0 để tự động ghi/kết thúc DB run |
| **Bulk Insert** | DatabaseManager.EndRun() | Gom dữ liệu RAM → ghi 1 transaction → nhanh ~100x |
| **Migration** | DatabaseManager.AddColumnIfMissing() | Tương thích ngược khi mở rộng schema DB |

---

## 📄 Tài liệu bổ sung

- 📐 [Kiến trúc phần mềm chi tiết (Tiếng Việt)](./docs/KIEN_TRUC_DU_AN.md)
- 🏗️ [Kiến trúc kỹ thuật](./docs/ARCHITECTURE.md)
