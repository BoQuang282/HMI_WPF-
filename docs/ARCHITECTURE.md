# 🏗️ KIẾN TRÚC PHẦN MỀM – HMI SCADA 3RRR ROBOT

> Tài liệu này mô tả chi tiết các quyết định thiết kế, luồng dữ liệu và pattern kiến trúc được áp dụng trong dự án HMI WPF.

---

## 1. Tổng quan Kiến trúc

Hệ thống được xây dựng theo mô hình **MVVM (Model – View – ViewModel)** kết hợp với **Singleton UartManager** để quản lý toàn bộ vòng đời giao tiếp phần cứng.

```
┌─────────────────────────────────────────────────────────────────┐
│                        PHẦN CỨNG (STM32)                        │
│                     UART @ 115200 baud                          │
└──────────────────────────────┬──────────────────────────────────┘
                               │ 101-byte Frame (30Hz)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   UartManager (Singleton)                        │
│  ┌─────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │ SerialPort  │→ │  ParseFrame()    │→ │  RobotData       │   │
│  │ Background  │  │  (101 bytes)     │  │  (Model)         │   │
│  │ Thread      │  └──────────────────┘  └────────┬─────────┘   │
│  └─────────────┘                                 │ DataReceived │
└──────────────────────────────────────────────────┼─────────────┘
                                                   │ Event (Thread-safe)
                               ┌───────────────────┴──────────────────┐
                               │                                       │
                               ▼                                       ▼
              ┌────────────────────────┐              ┌────────────────────────┐
              │   MainWindow.xaml.cs   │              │    Trends.xaml.cs      │
              │   (View + Glue Code)   │              │    (View + Render)     │
              │                        │              │                        │
              │  Dispatcher.Invoke()   │              │  DataLogger (ScottPlot)│
              │       ↓                │              │  DispatcherTimer 30fps │
              │  RobotViewModel        │              └────────────────────────┘
              │  .UpdateData(args)     │
              └────────────────────────┘
                          │
              ┌───────────▼───────────┐
              │    RobotViewModel     │
              │  (MVVM ViewModel)     │
              │                       │
              │  Observable Properties│
              │  StartCommand         │
              │  StopCommand          │
              │  HomeCommand          │
              └───────────────────────┘
```

---

## 2. Các Thành phần Cốt lõi

### 2.1. `UartManager` – Trung tâm Điều phối (Singleton)

**Vị trí:** `Robot/` (namespace `TestUart`)

`UartManager` là một **Singleton thread-safe** – chỉ tồn tại duy nhất 1 instance trong suốt vòng đời ứng dụng. Đây là lớp quan trọng nhất của hệ thống.

**Trách nhiệm:**
- Mở/đóng cổng COM (`SerialPort`)
- Lắng nghe dữ liệu đến trên **luồng nền (Background Thread)** – không chặn UI
- Phân tích khung dữ liệu 101 byte theo giao thức phần cứng
- Phát sự kiện `DataReceived` đến **tất cả** các View đang lắng nghe
- Phát sự kiện `TrajectoryStarted` khi nhận lệnh START để reset đồ thị
- Lưu trữ danh sách `XList`, `YList` cho quỹ đạo XY (dùng chung toàn ứng dụng)

**Tại sao Singleton?**
> Nhiều màn hình (MainWindow, Trends, Parameters...) cần cùng lúc nhận dữ liệu từ 1 cổng COM. Nếu mỗi màn hình tự mở cổng COM riêng thì sẽ bị xung đột và lỗi `UnauthorizedAccessException`. Singleton giải quyết hoàn toàn vấn đề này.

---

### 2.2. Mô hình MVVM

```
ViewModelBase (INotifyPropertyChanged)
    └─ RobotViewModel
            ├─ Properties: ToaDoX, ToaDoY, GocAlpha, GocTheta1..3, Momen1..3
            ├─ StartCommand (RelayCommand)
            ├─ StopCommand  (RelayCommand)
            └─ HomeCommand  (RelayCommand)
```

**`ViewModelBase.cs`:** Cài đặt `INotifyPropertyChanged`. Mọi thay đổi thuộc tính thông qua `SetProperty()` sẽ tự động thông báo UI cập nhật.

**`RelayCommand.cs`:** Bọc `Action` thành `ICommand`. Nút bấm trong XAML bind trực tiếp vào `Command="{Binding StartCommand}"` mà không cần code-behind.

**`RobotViewModel.cs`:** Cầu nối trung tâm – nhận dữ liệu từ `UartManager` qua phương thức `UpdateData()`, cung cấp dữ liệu cho UI qua Data Binding.

---

### 2.3. Luồng dữ liệu chi tiết (Data Flow)

#### Nhận dữ liệu từ phần cứng:
```
STM32 gửi 101 bytes
    → SerialPort.DataReceived (Background Thread)
    → UartManager.ParseFrame()
    → event DataReceived được kích hoạt
    → MainWindow.xaml.cs lắng nghe:
        Dispatcher.BeginInvoke(() => _robotViewModel.UpdateData(args.Data))
    → RobotViewModel.SetProperty() thông báo binding
    → TextBlock trên UI tự cập nhật
```

#### Gửi lệnh xuống phần cứng:
```
Người dùng bấm nút START
    → Button.Command = StartCommand (binding XAML)
    → RelayCommand.Execute()
    → RobotViewModel.StartCommand_Execute()
    → UartManager.Instance.SendControlSignal(0x01)
    → SerialPort.Write([0x01])
    → STM32 nhận lệnh
```

---

### 2.4. `Trends.xaml.cs` – Đồ thị thời gian thực

**Thư viện:** ScottPlot 5 (WPF wrapper `WpfPlot`)

**Cơ chế hoạt động:**
- `DispatcherTimer` chạy ở **30 FPS** (mỗi 33ms) trên UI Thread
- Mỗi tick: kiểm tra `SystemState` từ dữ liệu cuối cùng nhận được

**Logic đóng băng đồ thị:**
```csharp
bool dangChay = UartManager.Instance.IsConnected && _lastData.SystemState == 1;

if (dangChay)
{
    // Ghi điểm mới vào DataLogger
    // Cuộn trục X sang trái
}
// Nếu Stop/Home → không làm gì → đồ thị đứng yên
// Nếu Start mới → TrajectoryStarted event → Clear() toàn bộ Logger
```

**4 biểu đồ được duy trì:**

| Biểu đồ | Nội dung | Trục Y |
|---------|---------|--------|
| `plotTrajectory` | Quỹ đạo XY thực tế | Cố định `[0.15, 0.35] x [0.05, 0.25]` |
| `plotTheta` | Góc Theta 1, 2, 3 theo thời gian | Auto Scale |
| `plotTorque` | Momen động cơ 1, 2, 3 | Auto Scale |
| `plotError` | Sai số điều khiển 1, 2, 3 | Auto Scale |

---

### 2.5. Thread Safety (An toàn đa luồng)

Đây là vấn đề quan trọng nhất trong lập trình WPF với UART:

| Vấn đề | Giải pháp |
|--------|-----------|
| UART chạy trên Background Thread, không thể truy cập UI trực tiếp | `Dispatcher.Invoke()` / `BeginInvoke()` để chuyển sang UI Thread |
| Nhiều Thread cùng đọc/ghi `XList`, `YList` | `lock (UartManager.Instance.XList)` trước khi truy cập |
| ScottPlot Render cùng lúc ghi dữ liệu | `DataLogger` của ScottPlot 5 đã thread-safe nội bộ |

---

## 3. Giao diện người dùng

### Cấu trúc Layout

Toàn bộ ứng dụng dùng kiến trúc **Canvas cố định 1366×768px** bọc trong `Viewbox(Stretch="Uniform")`:

```
Window (AllowsTransparency, WindowStyle=None)
└─ Grid
   ├─ Row 0 (35px): Custom Title Bar
   │   ├─ TextBlock: Tên ứng dụng
   │   └─ StackPanel: [Thu nhỏ] [Phóng to] [✕ Đóng]
   └─ Row 1 (*): Viewbox (Stretch=Uniform)
       └─ Canvas (1366×768)
           ├─ Header (Logo + Tiêu đề + Đồng hồ)
           ├─ Nội dung chính (khác nhau mỗi màn hình)
           └─ NavBar (Thanh điều hướng 5 nút)
```

**Tại sao dùng `Viewbox`?**
> Khi người dùng phóng to màn hình (Maximize), Canvas 1366×768 sẽ được scale tỷ lệ đồng dạng (`Uniform`) lên đầy màn hình. Font chữ, nút bấm, đồ thị đều phóng to cùng tỷ lệ, không bị vỡ layout.

### Hệ thống Style (Resource Dictionary)

Mỗi màn hình định nghĩa 4 Style chuẩn trong `Window.Resources`:

| Style Key | Ứng dụng |
|-----------|---------|
| `HeaderTextStyle` | Tiêu đề ô panel (25px, Bold, Trắng) |
| `ContentTextStyle` | Dữ liệu số liệu (20px, Bold, Trắng) |
| `RoundedButtonStyle` | Nút bấm bo tròn với viền trắng |
| `PanelStyle` | Ô panel nền tối (HeaderedContentControl) |
| `TitleBarButtonStyle` | Nút trên thanh tiêu đề (hover xám) |
| `CloseButtonStyle` | Nút đóng (hover đỏ `#E81123`) |

---

## 4. Cấu hình phần cứng

### Robot Song song Phẳng 3RRR

| Thông số | Giá trị |
|---------|--------|
| Loại robot | 3RRR Parallel Planar |
| Số bậc tự do | 3 (X, Y, Alpha) |
| Chiều dài cánh tay chủ động (L1) | 0.20 m |
| Chiều dài cánh tay bị động (L2) | 0.20 m |
| Bán kính nội tiếp mâm (L3) | 0.125/√3 m |
| Không gian làm việc hiển thị | X: [0.15, 0.35] m, Y: [0.05, 0.25] m |

### Cấu hình UART

| Thông số | Giá trị |
|---------|--------|
| Baudrate | 115200 (mặc định) |
| Data bits | 8 |
| Stop bits | 1 |
| Parity | None |
| Tần suất dữ liệu | ~30Hz |
| Kích thước khung | 101 bytes |

---

## 5. Các quyết định thiết kế quan trọng

### Q: Tại sao không dùng `async/await` cho UART?
> `SerialPort.DataReceived` là sự kiện đặc biệt chạy trên **ThreadPool Thread**. Việc dùng `async` có thể gây deadlock với Dispatcher. Giải pháp hiện tại dùng `Dispatcher.BeginInvoke` (non-blocking) là an toàn và hiệu quả nhất cho WPF.

### Q: Tại sao giữ logic DragMove trong Code-Behind thay vì ViewModel?
> `DragMove()` là phương thức của lớp `Window` – thuần giao diện. Theo nguyên tắc MVVM, logic thuần UI (không liên quan business) được phép ở Code-Behind. Đây là quyết định đúng đắn.

### Q: Tại sao ScottPlot dùng `DataLogger` thay vì `Scatter` cho đồ thị thời gian?
> `DataLogger` của ScottPlot 5 được tối ưu cho việc thêm điểm liên tục theo thời gian thực. Nó không cần rebuild toàn bộ array mỗi frame, giúp đồ thị chạy ở 30fps mà không gây lag.

---

## 6. Hệ thống Ghi Log SQLite (Data Logging)

> **Gói NuGet:** `Microsoft.Data.Sqlite 8.0.0`  
> **File DB:** `robot_log.db` (tạo tự động cạnh file `.exe`)

### 6.1. Các thành phần mới

| File | Mô tả |
|------|-------|
| `Models/DbModels.cs` | Hai lớp C# ánh xạ đến bảng `RunHistory` và `RunData` |
| `Services/DatabaseManager.cs` | Singleton điều phối toàn bộ tương tác với SQLite |

### 6.2. Sơ đồ luồng dữ liệu Log

```
Người dùng nhấn START
    → RobotViewModel.ExecuteStart()
    → DatabaseManager.Instance.StartNewRun("Hình Vuông")
    → INSERT INTO RunHistory → trả về RunID = 5
    → _currentRunId = 5, _startTime = now, _currentRunData = new List()

Mỗi frame UART (~30Hz)
    → Dispatcher.Invoke → RobotViewModel.UpdateData(x, y, ...)
    → Tính TimeOffset = (now - _startTime).TotalSeconds
    → _currentRunData.Add(new RunData { RunID=5, TimeOffset=0.1, X=..., ... })
    → Dữ liệu tích lũy trong RAM

Người dùng nhấn STOP (hoặc xảy ra Fault)
    → RobotViewModel.SaveAndEndCurrentRun("Stop")
    → DatabaseManager.Instance.EndRun(5, "Stop", _currentRunData)
        ├── UPDATE RunHistory SET EndTime=now, EndStatus='Stop' WHERE RunID=5
        └── BEGIN TRANSACTION
            └── INSERT INTO RunData (loạt 1800 dòng) -- BULK INSERT
            COMMIT
    → _currentRunData.Clear() -- Giải phóng RAM
    → _currentRunId = -1
```

### 6.3. Cấu trúc bảng CSDL

#### Bảng `RunHistory`

| Cột | Kiểu | Mô tả |
|-----|------|-------|
| `RunID` | INTEGER PK | Tự tăng, ID lần chạy |
| `TrajName` | TEXT | Tên quỹ đạo: "Hình Vuông", "Chữ DUT"... |
| `StartTime` | TEXT | ISO 8601: `2026-04-15T09:00:00.000` |
| `EndTime` | TEXT | Thời điểm kết thúc |
| `EndStatus` | TEXT | `"Hoàn thành"` / `"Stop"` / `"Fault"` |

#### Bảng `RunData`

| Cột | Kiểu | Mô tả |
|-----|------|-------|
| `DataID` | INTEGER PK | Tự tăng |
| `RunID` | INTEGER FK | Tham chiếu `RunHistory.RunID` |
| `TimeOffset` | REAL | Giây kể từ StartTime |
| `Target_X/Y` | REAL | Setpoint vị trí mục tiêu (m) |
| `X, Y, Alpha` | REAL | Vị trí & định hướng thực tế |
| `Theta1/2/3` | REAL | Góc khớp (radian) |
| `Torque1/2/3` | REAL | Momen động cơ (N·m) |
| `Error1/2/3` | REAL | Sai số điều khiển |

### 6.4. Khởi tạo – Thêm 1 dòng vào `App.xaml.cs`

```csharp
// Trong App.xaml.cs – OnStartup override
protected override void OnStartup(StartupEventArgs e)
{
    base.OnStartup(e);
    // Tạo bảng SQLite nếu chưa tồn tại (chỉ chạy 1 lần khi khởi động)
    DatabaseManager.Instance.InitializeDatabase();
}
```

### 6.5. Lý do dùng Transaction cho Bulk Insert

> Không dùng Transaction: 1800 lần ghi = 1800 lần flush xuống đĩa → **rất chậm (~18 giây)**.  
> Dùng Transaction: Toàn bộ 1800 dòng được gom lại, **flush 1 lần duy nhất → ~0.1 giây**.  
> Kết quả: nhanh hơn **~100–200 lần**.
