# 📋 BÁO CÁO QUÁ TRÌNH THỰC HIỆN DỰ ÁN

# HỆ THỐNG ĐIỀU KHIỂN GIÁM SÁT ROBOT SONG SONG 3RRR

> **Đề tài:** Ứng dụng mạng nơ-ron nhân tạo RBFNN và thiết kế, thi công hệ thống điều khiển giám sát cho Robot song song phẳng 3 bậc tự do
>
> **Trường:** Đại học Bách khoa — Đại học Đà Nẵng · Khoa Điện · Lớp 21TDH2  
> **GVHD:** PGS. TS Lê Tiến Dũng  
> **SVTH:** Phan Hữu Anh Đức (105210311) · Trương Bảo Quang (105210332)
>
> **Ngày lập:** 22/04/2026

---

## 📑 MỤC LỤC

1. [Giới thiệu và Mục tiêu dự án](#chương-1-giới-thiệu-và-mục-tiêu-dự-án)
2. [Phân tích yêu cầu và Lựa chọn công nghệ](#chương-2-phân-tích-yêu-cầu-và-lựa-chọn-công-nghệ)
3. [Thiết kế kiến trúc phần mềm](#chương-3-thiết-kế-kiến-trúc-phần-mềm)
4. [Xây dựng tầng giao tiếp phần cứng (UART)](#chương-4-xây-dựng-tầng-giao-tiếp-phần-cứng)
5. [Xây dựng module Động học Robot](#chương-5-xây-dựng-module-động-học-robot)
6. [Xây dựng tầng xử lý logic (ViewModel & MVVM)](#chương-6-xây-dựng-tầng-xử-lý-logic)
7. [Xây dựng tầng dữ liệu (Database)](#chương-7-xây-dựng-tầng-dữ-liệu)
8. [Xây dựng giao diện người dùng (5 màn hình)](#chương-8-xây-dựng-giao-diện-người-dùng)
9. [Cơ chế hoạt động tổng thể — Luồng dữ liệu end-to-end](#chương-9-cơ-chế-hoạt-động-tổng-thể)
10. [Các vấn đề kỹ thuật gặp phải và cách giải quyết](#chương-10-các-vấn-đề-kỹ-thuật-và-cách-giải-quyết)
11. [Tổng kết và Đánh giá](#chương-11-tổng-kết-và-đánh-giá)

---

## CHƯƠNG 1: GIỚI THIỆU VÀ MỤC TIÊU DỰ ÁN

### 1.1. Bối cảnh

Dự án xây dựng phần mềm **HMI (Human-Machine Interface)** dạng **SCADA** để điều khiển và giám sát thời gian thực Robot Song song Phẳng 3RRR — một cơ cấu cơ khí 3 bậc tự do (x, y, α) được vận hành bởi 3 động cơ.

Hệ thống gồm 2 phần vật lý:
- **Vi điều khiển STM32:** Chạy thuật toán điều khiển RBFNN, đọc cảm biến, điều khiển 3 động cơ
- **Phần mềm HMI trên PC:** Nhận dữ liệu qua UART, hiển thị, tính toán động học, lưu lịch sử

### 1.2. Mục tiêu cụ thể

| STT | Mục tiêu | Mô tả chi tiết |
|-----|----------|-----------------|
| 1 | **Giao tiếp UART hai chiều** | Nhận 77 byte dữ liệu từ STM32 (~30 Hz), gửi 8 byte lệnh điều khiển |
| 2 | **Digital Twin 2D** | Mô phỏng robot thời gian thực trên Canvas: 3 cánh tay, mâm di động, quỹ đạo |
| 3 | **Tính toán Động học** | Tích hợp FK (Newton-Raphson) và IK (giải tích đóng) trực tiếp trên C# |
| 4 | **4 đồ thị thời gian thực** | Quỹ đạo XY, Theta, Momen, Sai số — 30 FPS |
| 5 | **Lưu trữ và tra cứu** | SQLite database lưu 24 cột dữ liệu/frame, xuất CSV, xóa theo bộ lọc |
| 6 | **Giám sát trạng thái** | LED chỉ thị Run/Stop/Fault, cảnh báo mất kết nối |

### 1.3. Điểm đặc biệt quan trọng

> ⚠️ **Sai số KHÔNG đến từ STM32.** STM32 chỉ gửi 9 giá trị mong muốn + 1 byte trạng thái. Toàn bộ giá trị **thực tế** (Actual) và **sai số** (Error) được **tính toán hoàn toàn trên C#** bằng Động học thuận (FK) và Động học nghịch (IK).

---

## CHƯƠNG 2: PHÂN TÍCH YÊU CẦU VÀ LỰA CHỌN CÔNG NGHỆ

### 2.1. Yêu cầu chức năng

```
Yêu cầu 1: Kết nối cổng COM và nhận/gửi dữ liệu nhị phân
Yêu cầu 2: Parse khung dữ liệu 77 byte với kiểm tra Header/Terminator
Yêu cầu 3: Tính toán Động học thuận/nghịch theo mô hình MATLAB gốc
Yêu cầu 4: Hiển thị mô phỏng robot 2D (Digital Twin)
Yêu cầu 5: Vẽ đồ thị thời gian thực (4 biểu đồ)
Yêu cầu 6: Bảng thông số kỹ thuật (24+ giá trị)
Yêu cầu 7: Lưu lịch sử vận hành vào cơ sở dữ liệu
Yêu cầu 8: Tra cứu, lọc, xuất CSV, xóa dữ liệu lịch sử
Yêu cầu 9: Hệ thống điều khiển Start/Stop/Home
Yêu cầu 10: Đăng nhập bảo mật
```

### 2.2. Yêu cầu phi chức năng

| Yêu cầu | Chỉ tiêu |
|----------|----------|
| **Hiệu suất** | Cập nhật giao diện ~30 FPS không bị giật lag |
| **An toàn luồng** | Đọc UART ở background thread, cập nhật UI ở UI thread |
| **Dữ liệu nhất quán** | Dùng Singleton đảm bảo 1 kết nối COM, 1 ViewModel xuyên suốt |
| **Dung lượng** | Ghi chạy 1 phút (~1800 điểm × 24 cột) trong < 100ms |
| **Tương thích ngược** | Database migration tự động thêm cột mới mà không mất dữ liệu cũ |

### 2.3. Lựa chọn công nghệ

| Thành phần | Công nghệ | Phiên bản | Lý do chọn |
|------------|-----------|-----------|-------------|
| **Ngôn ngữ** | C# | .NET 8.0 | Hỗ trợ tốt cho ứng dụng Windows desktop, hệ sinh thái mạnh |
| **Framework UI** | WPF | .NET 8 | Hỗ trợ Data Binding, MVVM, Canvas vẽ 2D, custom UI phong phú |
| **Đồ thị** | ScottPlot.WPF | 5.1.58 | Hiệu suất cao cho đồ thị thời gian thực, API dễ dùng |
| **Cơ sở dữ liệu** | Microsoft.Data.Sqlite | 8.0.0 | Nhẹ, không cần server, file-based (robot_log.db) |
| **Giao tiếp Serial** | System.IO.Ports | 10.0.5 | Thư viện chính thức cho UART/COM trên .NET |
| **Kiến trúc** | MVVM | — | Tách biệt giao diện và logic, dễ bảo trì và mở rộng |
| **IDE** | Visual Studio 2022 | — | Hỗ trợ tốt nhất cho WPF + .NET 8 |

### 2.4. Cấu hình dự án (`HMI_WPF.csproj`)

File `.csproj` định nghĩa cấu hình build và các thư viện phụ thuộc:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>          <!-- Ứng dụng Windows -->
    <TargetFramework>net8.0-windows</TargetFramework>  <!-- .NET 8 -->
    <UseWPF>true</UseWPF>                    <!-- Bật WPF -->
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.Data.Sqlite" Version="8.0.0" />
    <PackageReference Include="ScottPlot.WPF" Version="5.1.58" />
    <PackageReference Include="System.IO.Ports" Version="10.0.5" />
  </ItemGroup>
</Project>
```

**Giải thích:**
- `OutputType = WinExe`: Biên dịch thành ứng dụng Windows có cửa sổ (không phải console)
- `TargetFramework = net8.0-windows`: Framework đích là .NET 8 với hỗ trợ Windows-specific APIs
- `UseWPF = true`: Cho phép sử dụng WPF framework (XAML, Canvas, Data Binding...)
- 3 gói NuGet là các thư viện bên thứ ba được cài qua Package Manager

---

## CHƯƠNG 3: THIẾT KẾ KIẾN TRÚC PHẦN MỀM

### 3.1. Kiến trúc phân tầng

Ứng dụng được thiết kế theo kiến trúc **4 tầng (4-tier)**, tách biệt rõ ràng trách nhiệm:

```
┌─────────────────────────────────────────────────────────┐
│  TẦNG 1: GIAO DIỆN (Presentation Layer)                │
│  Login.xaml · MainWindow.xaml · Trends.xaml              │
│  Parameters.xaml · History.xaml                           │
│  → Hiển thị UI, nhận tương tác người dùng               │
├─────────────────────────────────────────────────────────┤
│  TẦNG 2: XỬ LÝ LOGIC (Business Logic Layer)            │
│  RobotViewModel.cs · ViewModelBase.cs · RelayCommand.cs │
│  → Edge Detection, Start/Stop, Data Binding, DB logging │
├─────────────────────────────────────────────────────────┤
│  TẦNG 3: GIAO TIẾP & TÍNH TOÁN (Communication Layer)   │
│  UartManager.cs · RobotData.cs · RobotKinematics.cs     │
│  → Đọc/ghi UART, parse frame, FK/IK, quản lý danh sách │
├─────────────────────────────────────────────────────────┤
│  TẦNG 4: DỮ LIỆU (Data Layer)                          │
│  DatabaseManager.cs · DbModels.cs · robot_log.db        │
│  → CRUD SQLite, Bulk Insert, Migration                   │
└─────────────────────────────────────────────────────────┘
```

### 3.2. Sơ đồ cấu trúc file dự án

```
HMI_WPF/
│
├── HMI_WPF.slnx               ← File Solution (mở bằng Visual Studio)
├── README.md                   ← Tóm tắt dự án
│
├── docs/                       ← 📚 Tài liệu kỹ thuật
│   ├── KIEN_TRUC_DU_AN.md     ← Kiến trúc chi tiết
│   ├── BAO_CAO_QUA_TRINH_THUC_HIEN.md  ← File này
│   └── ARCHITECTURE.md        ← Kiến trúc kỹ thuật (Tiếng Anh)
│
└── HMI_WPF/                   ← 📂 Mã nguồn chính
    │
    ├── App.xaml + .cs          ← Điểm vào ứng dụng
    ├── Login.xaml + .cs        ← Màn hình đăng nhập
    ├── MainWindow.xaml + .cs   ← Trang chủ SCADA
    ├── Trends.xaml + .cs       ← Đồ thị thời gian thực
    ├── Parameters.xaml + .cs   ← Bảng thông số
    ├── History.xaml + .cs      ← Lịch sử vận hành
    │
    ├── Robot/                  ← Tầng giao tiếp + Tính toán
    │   ├── RobotData.cs        ← Mô hình dữ liệu
    │   ├── UartManager.cs      ← Quản lý UART (Singleton)
    │   └── RobotKinematics.cs  ← FK + IK
    │
    ├── ViewModels/             ← Tầng xử lý logic (MVVM)
    │   ├── ViewModelBase.cs    ← INotifyPropertyChanged
    │   ├── RelayCommand.cs     ← ICommand wrapper
    │   └── RobotViewModel.cs   ← ViewModel chính
    │
    ├── Models/                 ← Cấu trúc CSDL
    │   └── DbModels.cs         ← RunHistory + RunData
    │
    └── Services/               ← Dịch vụ
        └── DatabaseManager.cs  ← SQLite Singleton
```

### 3.3. Các mẫu thiết kế (Design Patterns) được áp dụng

| Pattern | Nơi áp dụng | Mục đích | Cách hoạt động |
|---------|-------------|----------|----------------|
| **Singleton** | UartManager, DatabaseManager, SharedRobotViewModel | Đảm bảo chỉ tồn tại 1 instance duy nhất trong toàn bộ ứng dụng | Khai báo `public static readonly Instance = new()` hoặc `public static ... { get; } = new()` |
| **MVVM** | ViewModelBase + RobotViewModel + View | Tách biệt UI (XAML) khỏi logic xử lý (C#) | View bind đến ViewModel qua `{Binding PropertyName}`, ViewModel kế thừa ViewModelBase |
| **Observer/Event** | DataReceived, TrajectoryStarted, RunStarted | Khi dữ liệu đến, broadcast tự động đến tất cả người nghe | Sử dụng `event EventHandler<T>`, các trang giao diện đăng ký bằng `+=` |
| **Edge Detection** | RobotViewModel.UpdateData() | Phát hiện khi SystemState chuyển từ 0→1 (sườn lên) hoặc 1→0 (sườn xuống) | So sánh `_previousSystemState` với `currentSystemState` mỗi frame |
| **Bulk Insert** | DatabaseManager.EndRun() | Gom hàng nghìn điểm dữ liệu trong RAM, ghi 1 lần vào DB khi Stop | Sử dụng SQLite Transaction cho toàn bộ INSERT |
| **Migration** | DatabaseManager.AddColumnIfMissing() | Thêm cột mới vào DB cũ mà không mất dữ liệu | `ALTER TABLE ... ADD COLUMN ...` bọc trong try-catch |

---

## CHƯƠNG 4: XÂY DỰNG TẦNG GIAO TIẾP PHẦN CỨNG

### 4.1. Tổng quan module `UartManager.cs`

**File:** `Robot/UartManager.cs` (352 dòng)  
**Vai trò:** Quản lý toàn bộ giao tiếp UART với STM32 — nhận dữ liệu, gửi lệnh, tính toán Động học, lưu trữ vào danh sách Singleton.

**Mẫu thiết kế:** Singleton — chỉ có đúng 1 instance trong toàn bộ ứng dụng.

```csharp
public class UartManager : IDisposable
{
    // Singleton: Tạo duy nhất 1 lần, truy cập từ mọi nơi bằng UartManager.Instance
    public static readonly UartManager Instance = new UartManager();
    
    // Constructor private → Ngăn không cho tạo instance mới từ bên ngoài
    private UartManager() { }
}
```

**Tại sao dùng Singleton?** Vì chỉ được mở 1 cổng COM tại 1 thời điểm. Nếu có nhiều instance, sẽ gây lỗi "COM port đã bị chiếm". Tất cả 5 trang giao diện đều dùng chung `UartManager.Instance`.

### 4.2. Các biến trạng thái quan trọng

```csharp
// ── Kết nối Serial ────────────────────────────────
private SerialPort?     _serialPort;      // Đối tượng đại diện cổng COM
private Thread?         _readThread;       // Luồng nền đọc dữ liệu liên tục
private volatile bool   _shouldContinue;   // Cờ kiểm soát vòng lặp đọc
private readonly object _writeLock = new(); // Khóa đồng bộ khi ghi xuống COM

// ── Bộ đệm ghép khung ────────────────────────────
private byte[]? _currentFrame;      // Mảng 77 byte chứa khung đang ghép
private int     _frameByteIndex;    // Đang ghép đến byte thứ mấy (0..76)
private int     _headerCount;       // Đã đếm được bao nhiêu byte header 0x7E liên tiếp

// ── Hằng số giao thức ─────────────────────────────
private const byte HEADER_BYTE      = 0x7E;  // = 126
private const byte TERMINATOR_BYTE  = 0x03;  // = 3
private const int  FRAME_TOTAL_SIZE = 77;     // Header(2) + Data(72) + State(1) + Term(2)
```

**Giải thích từng biến:**
- `_serialPort`: Đối tượng `SerialPort` của .NET, đại diện cho kết nối USB-UART. Khi gọi `_serialPort.Open()`, hệ điều hành sẽ dành riêng cổng COM đó cho ứng dụng.
- `_readThread`: Một luồng (Thread) chạy nền, liên tục đọc từng byte từ cổng COM. Chạy song song với luồng giao diện (UI Thread) để không gây giật lag.
- `_shouldContinue` (volatile): Cờ boolean kiểm soát vòng lặp. Khi đặt `false`, luồng đọc tự dừng. Từ khóa `volatile` đảm bảo tất cả các luồng đọc giá trị mới nhất.
- `_writeLock`: Khóa mutex — khi cần ghi dữ liệu xuống COM, dùng `lock(_writeLock)` để đảm bảo chỉ 1 luồng ghi tại 1 thời điểm.
- `_currentFrame`: Mảng byte[77] đóng vai trò "tấm bảng đang viết dở" — mỗi khi nhận 1 byte mới, ghi vào vị trí `_frameByteIndex` trên tấm bảng này.
- `_headerCount`: Đếm byte header. Khi đếm được 2 byte `0x7E` liên tiếp, bắt đầu ghép khung mới.

### 4.3. Danh sách dữ liệu Singleton (21 danh sách)

UartManager lưu trữ **toàn bộ dữ liệu** đã nhận kể từ lần Start gần nhất. Mỗi danh sách là một `List<double>` tương ứng với 1 kênh dữ liệu:

```csharp
// ── Mong muốn (từ UART) ─────────────
public List<double> TimeList    = new();  // Nhãn thời gian (giây)
public List<double> XList       = new();  // PosX mong muốn
public List<double> YList       = new();  // PosY mong muốn
public List<double> Theta1List  = new();  // Theta1 mong muốn
public List<double> Theta2List  = new();
public List<double> Theta3List  = new();
public List<double> Torque1List = new();  // Momen 3 động cơ
public List<double> Torque2List = new();
public List<double> Torque3List = new();

// ── Thực tế (từ FK/IK – tính trên C#) ───
public List<double> ActualXList      = new();  // FK → XY thực tế
public List<double> ActualYList      = new();
public List<double> ActualAlphaList  = new();
public List<double> ActualTheta1List = new();  // IK → Theta thực tế
public List<double> ActualTheta2List = new();
public List<double> ActualTheta3List = new();

// ── Sai số (tính trên C#) ───────────────
public List<double> ErrorXList      = new();
public List<double> ErrorYList      = new();
public List<double> ErrorAlphaList  = new();
public List<double> ErrorTheta1List = new();
public List<double> ErrorTheta2List = new();
public List<double> ErrorTheta3List = new();
```

**Tại sao lưu trong UartManager?** Vì các trang giao diện (Trends, Parameters) có thể bị tạo/hủy khi chuyển trang, nhưng UartManager tồn tại suốt vòng đời ứng dụng → dữ liệu không bao giờ bị mất.

### 4.4. Quy trình kết nối và đọc dữ liệu

#### Bước 1: Khởi động kết nối (`Start()`)

```csharp
public void Start(string portName, int baudRate)
{
    if (IsConnected) return;  // Đã kết nối rồi → bỏ qua

    _serialPort = new SerialPort(portName, baudRate, Parity.None, 8, StopBits.One)
    {
        ReadTimeout  = 100,   // Timeout đọc = 100ms
        WriteTimeout = 1000,  // Timeout ghi = 1000ms
        DtrEnable    = true,  // Bật tín hiệu DTR (Data Terminal Ready)
        RtsEnable    = true   // Bật tín hiệu RTS (Request To Send)
    };

    _serialPort.Open();           // Mở cổng COM (chiếm hữu cổng)
    _shouldContinue = true;       // Cho phép vòng lặp đọc chạy
    _currentFrame   = new byte[FRAME_TOTAL_SIZE];  // Tạo bộ đệm 77 byte

    // Tạo luồng nền đọc dữ liệu
    _readThread = new Thread(ReadThread)
    {
        IsBackground = true,      // Tự tắt khi ứng dụng đóng
        Name = "UART Read Thread"  // Đặt tên cho dễ debug
    };
    _readThread.Start();          // Bắt đầu đọc
}
```

**Giải thích chi tiết:**
- `DtrEnable = true, RtsEnable = true`: Bật 2 đường tín hiệu điều khiển trên cổng COM. Nhiều vi điều khiển (bao gồm STM32) yêu cầu DTR phải bật mới chấp nhận kết nối.
- `IsBackground = true`: Đánh dấu luồng đọc là luồng nền. Khi tất cả luồng foreground kết thúc (tức ứng dụng đóng), các luồng background tự động bị hủy — tránh ứng dụng "zombie" (không tắt được).

#### Bước 2: Vòng lặp đọc byte (`ReadThread()`)

```csharp
private void ReadThread()
{
    while (_shouldContinue && _serialPort != null && _serialPort.IsOpen)
    {
        try
        {
            if (_serialPort.BytesToRead > 0)      // Có dữ liệu mới?
                ProcessByte((byte)_serialPort.ReadByte());  // Đọc 1 byte, xử lý
            else
                Thread.Sleep(1);  // Không có dữ liệu → nghỉ 1ms (tránh CPU 100%)
        }
        catch (TimeoutException) { Thread.Sleep(1); }  // Timeout → bỏ qua
        catch { break; }  // Lỗi nghiêm trọng → thoát vòng lặp
    }
}
```

**Tại sao đọc từng byte?** Vì giao tiếp UART truyền từng byte một. Dữ liệu có thể đến "giọt giọt" (vài byte mỗi lần) hoặc "ào ào" (cả khung cùng lúc). Đọc từng byte giúp xử lý đúng mọi trường hợp.

#### Bước 3: Ghép khung từ byte đơn lẻ (`ProcessByte()`)

Đây là **máy trạng thái (state machine)** ghép từng byte thành khung 77 byte hoàn chỉnh:

```csharp
private void ProcessByte(byte b)
{
    // ── TRẠNG THÁI 1: Đang tìm Header (0x7E 0x7E) ──
    if (_frameByteIndex < 2)
    {
        if (b == HEADER_BYTE)       // Nhận được 0x7E?
        {
            _headerCount++;         // Tăng bộ đếm header
            if (_headerCount == 2)  // Đã đếm đủ 2 byte 0x7E liên tiếp?
            {
                // → Tìm thấy Header! Bắt đầu ghép khung mới
                _currentFrame[0] = HEADER_BYTE;
                _currentFrame[1] = HEADER_BYTE;
                _frameByteIndex  = 2;   // Đã ghi 2 byte đầu
                _headerCount     = 0;
            }
        }
        else
        {
            // Byte không phải 0x7E → reset bộ đếm
            _headerCount = b == HEADER_BYTE ? 1 : 0;
        }
    }
    // ── TRẠNG THÁI 2: Đang ghép thân khung (byte 2..76) ──
    else if (_frameByteIndex < FRAME_TOTAL_SIZE)
    {
        _currentFrame[_frameByteIndex] = b;   // Ghi byte vào bộ đệm
        _frameByteIndex++;

        if (_frameByteIndex == FRAME_TOTAL_SIZE)   // Đủ 77 byte?
        {
            // Kiểm tra Terminator (2 byte cuối phải là 0x03 0x03)
            if (_currentFrame[75] == TERMINATOR_BYTE && 
                _currentFrame[76] == TERMINATOR_BYTE)
            {
                ProcessCompleteFrame();  // Khung hợp lệ → xử lý!
            }
            // Reset cho khung tiếp theo
            _frameByteIndex = 0;
            _headerCount    = 0;
        }
    }
}
```

**Hình ảnh hóa quá trình:**
```
Dòng byte đến:  ... [rác] [0x7E] [0x7E] [72 byte data] [state] [0x03] [0x03] [rác] ...
                           ↑ Header found!                                ↑ Terminator OK!
                           `_frameByteIndex = 2`                          `ProcessCompleteFrame()`
```

#### Bước 4: Xử lý khung hoàn chỉnh (`ProcessCompleteFrame()`)

Đây là **trái tim** của hệ thống — hàm được gọi ~30 lần/giây, thực hiện 6 bước:

```csharp
private void ProcessCompleteFrame()
{
    // ═══ Bước 1: Parse dữ liệu thô từ byte[] → đối tượng RobotData ═══
    var robotData = new RobotData
    {
        // Mỗi giá trị double chiếm 8 byte, dùng BitConverter để chuyển đổi
        Theta1 = BitConverter.ToDouble(_currentFrame, 2),    // byte[2..9]
        Theta2 = BitConverter.ToDouble(_currentFrame, 10),   // byte[10..17]
        Theta3 = BitConverter.ToDouble(_currentFrame, 18),   // byte[18..25]
        PosX   = BitConverter.ToDouble(_currentFrame, 26),   // byte[26..33]
        PosY   = BitConverter.ToDouble(_currentFrame, 34),   // byte[34..41]
        Alpha  = BitConverter.ToDouble(_currentFrame, 42),   // byte[42..49]
        Torque1 = BitConverter.ToDouble(_currentFrame, 50),  // byte[50..57]
        Torque2 = BitConverter.ToDouble(_currentFrame, 58),  // byte[58..65]
        Torque3 = BitConverter.ToDouble(_currentFrame, 66),  // byte[66..73]
        SystemState  = _currentFrame[74],                    // byte[74] = 1/0
        ReceivedTime = DateTime.Now                           // Gắn timestamp
    };

    // ═══ Bước 2: Tính Động học thuận FK ═══
    // Input:  Theta mong muốn (từ UART) + initial guess (XYAlpha mong muốn)
    // Output: XY Alpha THỰC TẾ
    RobotKinematics.ForwardKinematics(
        robotData.Theta1, robotData.Theta2, robotData.Theta3,
        robotData.PosX, robotData.PosY, robotData.Alpha,    // initial guess
        out double actualX, out double actualY, out double actualAlpha);

    robotData.ActualX     = actualX;
    robotData.ActualY     = actualY;
    robotData.ActualAlpha = actualAlpha;

    // ═══ Bước 3: Tính Động học nghịch IK ═══
    // Input:  XYAlpha mong muốn (từ UART)
    // Output: Theta THỰC TẾ
    RobotKinematics.InverseKinematics(
        robotData.PosX, robotData.PosY, robotData.Alpha,
        out double actualTheta1, out double actualTheta2, out double actualTheta3);

    robotData.ActualTheta1 = actualTheta1;
    robotData.ActualTheta2 = actualTheta2;
    robotData.ActualTheta3 = actualTheta3;

    // ═══ Bước 4: Tính sai số = Mong muốn – Thực tế ═══
    robotData.ErrorX      = robotData.PosX  - actualX;
    robotData.ErrorY      = robotData.PosY  - actualY;
    robotData.ErrorAlpha  = robotData.Alpha - actualAlpha;
    robotData.ErrorTheta1 = robotData.Theta1 - actualTheta1;
    robotData.ErrorTheta2 = robotData.Theta2 - actualTheta2;
    robotData.ErrorTheta3 = robotData.Theta3 - actualTheta3;

    // ═══ Bước 5: Lưu vào 21 danh sách (thread-safe) ═══
    lock (XList)   // Khóa để tránh xung đột với Trends đang đọc
    {
        CurrentTime += 0.01;   // Tăng bộ đếm thời gian
        TimeList.Add(CurrentTime);
        XList.Add(robotData.PosX);
        YList.Add(robotData.PosY);
        // ... (21 danh sách, mỗi danh sách thêm 1 phần tử)
    }

    // ═══ Bước 6: Phát sóng sự kiện DataReceived ═══
    DataReceived?.Invoke(this, new RobotDataEventArgs(robotData));
    // → MainWindow nhận → cập nhật Digital Twin + ViewModel
    // → Parameters nhận → cập nhật bảng số
}
```

**Tại sao dùng `lock(XList)`?**  
Vì `ProcessCompleteFrame()` chạy ở **background thread** (luồng UART), trong khi Trends đọc `XList` ở **UI thread** (luồng giao diện). Nếu không lock, có thể xảy ra "race condition" — Trends đọc danh sách đang bị thêm phần tử → crash.

### 4.5. Gửi lệnh điều khiển (`SendControlSignal()`)

Khi người dùng nhấn Start/Stop/Home, phần mềm gửi **8 byte** xuống STM32:

```csharp
public void SendControlSignal(byte trajectoryId, bool isStart, bool isStop, bool isHome)
{
    if (!IsConnected) return;   // Chưa kết nối → bỏ qua

    // Tạo khung 8 byte
    byte[] frame = new byte[8];
    frame[0] = 0x7E;   // Header byte 1
    frame[1] = 0x7E;   // Header byte 2
    frame[2] = trajectoryId;           // ID quỹ đạo (1-9)
    frame[3] = (byte)(isStart ? 1 : 0);  // Start flag
    frame[4] = (byte)(isStop  ? 1 : 0);  // Stop flag
    frame[5] = (byte)(isHome  ? 1 : 0);  // Home flag
    frame[6] = 0x03;   // Terminator byte 1
    frame[7] = 0x03;   // Terminator byte 2

    lock (_writeLock)   // Đảm bảo chỉ 1 luồng ghi tại 1 thời điểm
    {
        _serialPort!.Write(frame, 0, frame.Length);
    }

    if (isStart)
    {
        ClearData();   // Xóa tất cả 21 danh sách dữ liệu
        TrajectoryStarted?.Invoke(this, EventArgs.Empty);  // Thông báo Trends
    }
}
```

**Lưu ý quan trọng:** Khi gửi lệnh Start, ngoài việc gửi byte xuống STM32, còn gọi `ClearData()` để xóa tất cả dữ liệu cũ trong 21 danh sách, và phát sự kiện `TrajectoryStarted` để Trends xóa đồ thị cũ.

### 4.6. Mô hình dữ liệu `RobotData.cs`

**File:** `Robot/RobotData.cs` (140 dòng)  
**Vai trò:** Lớp C# chứa **24+ thuộc tính** đại diện cho 1 frame dữ liệu hoàn chỉnh.

```csharp
public class RobotData
{
    // ── Từ UART (9 giá trị double + 1 byte trạng thái) ──
    public double Theta1 { get; set; }    // Góc mong muốn khớp 1 (rad)
    public double Theta2 { get; set; }    // Góc mong muốn khớp 2
    public double Theta3 { get; set; }    // Góc mong muốn khớp 3
    public double PosX   { get; set; }    // Tọa độ X mong muốn (m)
    public double PosY   { get; set; }    // Tọa độ Y mong muốn (m)
    public double Alpha  { get; set; }    // Góc nghiêng mong muốn (rad)
    public double Torque1 { get; set; }   // Momen động cơ 1 (N·m)
    public double Torque2 { get; set; }
    public double Torque3 { get; set; }
    public byte SystemState { get; set; } // 1=Run, 0=Stop
    public DateTime ReceivedTime { get; set; }

    // ── Từ FK (tính trên C#) ──
    public double ActualX { get; set; }      // X thực tế
    public double ActualY { get; set; }      // Y thực tế
    public double ActualAlpha { get; set; }  // Alpha thực tế

    // ── Từ IK (tính trên C#) ──
    public double ActualTheta1 { get; set; }
    public double ActualTheta2 { get; set; }
    public double ActualTheta3 { get; set; }

    // ── Sai số (tính trên C#) ──
    public double ErrorX { get; set; }       // = PosX - ActualX
    public double ErrorY { get; set; }
    public double ErrorAlpha { get; set; }
    public double ErrorTheta1 { get; set; }  // = Theta1 - ActualTheta1
    public double ErrorTheta2 { get; set; }
    public double ErrorTheta3 { get; set; }

    // Alias tương thích code cũ
    public double Error1 { get => ErrorTheta1; set => ErrorTheta1 = value; }
    public double Error2 { get => ErrorTheta2; set => ErrorTheta2 = value; }
    public double Error3 { get => ErrorTheta3; set => ErrorTheta3 = value; }
}
```

**`RobotDataEventArgs`** — Lớp bao bọc để truyền qua hệ thống sự kiện:
```csharp
public class RobotDataEventArgs : EventArgs
{
    public RobotData Data { get; }
    public RobotDataEventArgs(RobotData data) { Data = data; }
}
```

Giống như một **phong bì** chứa **bức thư** (RobotData). Hệ thống event của .NET yêu cầu tham số phải kế thừa `EventArgs` → cần lớp bao bọc này.

---

## CHƯƠNG 5: XÂY DỰNG MODULE ĐỘNG HỌC ROBOT

### 5.1. Tổng quan `RobotKinematics.cs`

**File:** `Robot/RobotKinematics.cs` (227 dòng)  
**Vai trò:** Module tính toán thuần số học, dịch trực tiếp từ mã MATLAB gốc. Không phụ thuộc vào giao diện hay UART.

**Từ khóa `static`:** Tất cả hàm đều là `static` — không cần tạo đối tượng, gọi trực tiếp `RobotKinematics.ForwardKinematics(...)`.

### 5.2. Thông số hình học cố định

```csharp
private const double L1 = 0.20;    // Cánh tay chủ động = 20cm
private const double L2 = 0.20;    // Cánh tay bị động = 20cm
private const double L3 = 0.125 / 1.7320508075688772;  // = 0.125/√3 ≈ 0.0722m

// Tọa độ 3 gốc động cơ trong hệ cố định (hệ mét)
private static readonly double[] Xo = { 0.0, 0.5, 0.25 };
private static readonly double[] Yo = { 0.0, 0.0, 0.5 * 1.7320508075688772 / 2.0 };

// Góc offset 3 chốt trên mâm di động
private static readonly double[] Gamma = { π/6, 5π/6, 3π/2 };

// Cấu hình Newton-Raphson
private const int    MAX_ITER = 300;    // Tối đa 300 vòng lặp
private const double EPSILON  = 1e-12;  // Ngưỡng hội tụ
```

### 5.3. Động học Nghịch (Inverse Kinematics) — Giải tích đóng

**Mục đích:** Cho biết vị trí mâm (x, y, α) → Tính góc mà các khớp cần có.

```
Đầu vào:  (x, y, alpha) — vị trí + hướng mâm MONG MUỐN
Đầu ra:   (theta1, theta2, theta3) — góc khớp THỰC TẾ
```

**Thuật toán (cho mỗi khớp i = 1, 2, 3):**

```
1. Tính góc phi — góc khớp bám trên mâm, phụ thuộc alpha:
   phi = gamma_base[i] + alpha

2. Tính tọa độ điểm bám B_i trên mâm:
   xc = x + L3·cos(phi) - Xo[i]
   yc = y + L3·sin(phi) - Yo[i]

3. Tính góc hướng si:
   si = atan2(yc, xc)

4. Tính khoảng cách c giữa gốc động cơ và điểm bám:
   c² = xc² + yc²

5. Tính góc beta bằng định lý cosine:
   cosArg = (L1² - L2² + c²) / (2·L1·c)
   cosArg = clamp(cosArg, -1, 1)     ← Tránh lỗi acos ngoài miền
   beta = acos(cosArg)

6. Tính Theta:
   theta[i] = si + beta
   Nếu theta > 2π → trừ 2π (chuẩn hóa về [0, 2π])
```

**Code C# tương ứng:**
```csharp
public static bool InverseKinematics(double x, double y, double alpha,
    out double theta1, out double theta2, out double theta3)
{
    for (int i = 0; i < 3; i++)
    {
        double phi = (i == 0 ? 7π/6 : i == 1 ? 11π/6 : π/2) + alpha;
        double xc = x + L3 * Math.Cos(phi) - Xo[i];
        double yc = y + L3 * Math.Sin(phi) - Yo[i];
        double si = Math.Atan2(yc, xc);
        double c2p = xc*xc + yc*yc;
        double c = Math.Sqrt(c2p);
        double cosArg = (L1*L1 - L2*L2 + c2p) / (2.0 * L1 * c);
        cosArg = Math.Max(-1.0, Math.Min(1.0, cosArg));  // Clamp
        double beta = Math.Acos(cosArg);
        theta[i] = si + beta;
    }
}
```

### 5.4. Động học Thuận (Forward Kinematics) — Newton-Raphson

**Mục đích:** Cho biết góc 3 khớp → Tính vị trí thực tế của mâm.

```
Đầu vào:  (theta1, theta2, theta3) — góc MONG MUỐN từ UART
           (x0, y0, alpha0) — initial guess = XYAlpha mong muốn
Đầu ra:   (actualX, actualY, actualAlpha) — XYAlpha THỰC TẾ
```

**Thuật toán Newton-Raphson (lặp):**

```
1. Khởi tạo: p = [x0, y0, alpha0]    ← điểm xuất phát (guess)
2. Lặp tối đa 300 vòng:
   a) Tính F(p)  — hệ 3 phương trình ràng buộc hình học
   b) Tính J(p)  — ma trận Jacobian 3×3
   c) Giải hệ:   J·δ = F   (bằng Gaussian elimination)
   d) Cập nhật:   p = p - δ
   e) Kiểm tra:   |δ| < ε  → dừng (đã hội tụ)
3. Kết quả: p = [actualX, actualY, actualAlpha]
```

**Hệ phương trình ràng buộc F(p):** Mỗi phương trình kiểm tra "cánh tay bị động có đúng chiều dài L2 không":

```csharp
F[i] = |B_i - A_i|² - L2²

Trong đó:
  A_i = O_i + L1·[cos(theta_i), sin(theta_i)]   ← đầu mút cánh tay chủ động
  B_i = [x, y] + L3·[cos(γ_i + α), sin(γ_i + α)]  ← chốt bám trên mâm
```

**Giải hệ tuyến tính 3×3:** Dùng phương pháp **Gaussian Elimination có Partial Pivoting** — an toàn hơn Cramer rule, tránh chia cho 0 khi ma trận gần suy biến.

```csharp
private static double[] SolveLinear3x3(double[,] A, double[] b)
{
    // 1. Tạo ma trận mở rộng [A|b] kích thước 3×4
    double[,] M = new double[3, 4];
    
    for (int col = 0; col < 3; col++)
    {
        // 2. Tìm hàng có phần tử lớn nhất (pivoting) → tránh chia cho số nhỏ
        int maxRow = col;
        for (int row = col+1; row < 3; row++)
            if (Math.Abs(M[row,col]) > Math.Abs(M[maxRow,col]))
                maxRow = row;
        
        // 3. Hoán đổi hàng
        (M[col], M[maxRow]) = swap;
        
        // 4. Khử Gauss: trừ hàng col cho các hàng bên dưới
        for (int row = col+1; row < 3; row++)
            M[row] -= factor * M[col];
    }
    
    // 5. Thế ngược (back substitution)
    for (int i = 2; i >= 0; i--)
        x[i] = (M[i,3] - Σ M[i,j]*x[j]) / M[i,i];
    
    return x;
}
```

---

## CHƯƠNG 6: XÂY DỰNG TẦNG XỬ LÝ LOGIC

### 6.1. Nền tảng MVVM — `ViewModelBase.cs`

**File:** `ViewModels/ViewModelBase.cs` (34 dòng)  
**Vai trò:** Lớp cơ sở trừu tượng cho tất cả ViewModel, triển khai giao diện `INotifyPropertyChanged`.

```csharp
public abstract class ViewModelBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    // Phát tín hiệu "thuộc tính X đã thay đổi" → WPF tự động cập nhật giao diện
    protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }

    // Gán giá trị + tự động phát tín hiệu nếu giá trị thực sự đổi
    protected bool SetProperty<T>(ref T field, T value, 
        [CallerMemberName] string? propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value))
            return false;       // Giá trị không đổi → không phát tín hiệu (tối ưu)

        field = value;          // Gán giá trị mới
        OnPropertyChanged(propertyName);  // Phát tín hiệu
        return true;
    }
}
```

**Cách hoạt động của Data Binding:**
```
XAML:   <TextBlock Text="{Binding ToaDoX, StringFormat=F3}" />
                                 ↑
                                 │ Binding đến property
                                 │
C#:     public double ToaDoX     │
        {                        │
            get => _toaDoX;      │
            set => SetProperty(ref _toaDoX, value);  // Khi set mới → phát PropertyChanged
        }                                             //   → WPF bắt tín hiệu → cập nhật TextBlock
```

### 6.2. Cơ chế lệnh — `RelayCommand.cs`

**File:** `ViewModels/RelayCommand.cs` (40 dòng)  
**Vai trò:** Cho phép gắn hàm C# trực tiếp vào nút bấm XAML thông qua Data Binding.

```csharp
public class RelayCommand : ICommand
{
    private readonly Action<object?> _execute;         // Hàm thực thi khi nhấn nút
    private readonly Func<object?, bool>? _canExecute; // Hàm kiểm tra: nút có được bật không?

    public event EventHandler? CanExecuteChanged
    {
        add { CommandManager.RequerySuggested += value; }
        remove { CommandManager.RequerySuggested -= value; }
    }

    public RelayCommand(Action<object?> execute, Func<object?, bool>? canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    public bool CanExecute(object? parameter) => _canExecute == null || _canExecute(parameter);
    public void Execute(object? parameter) => _execute(parameter);
}
```

**Ví dụ sử dụng:**
```csharp
// Trong RobotViewModel:
StartCommand = new RelayCommand(ExecuteStart);  // Gắn hàm ExecuteStart vào StartCommand

// Trong XAML:
// <Button Content="START" Command="{Binding StartCommand}" />
// → Khi nhấn nút → WPF gọi StartCommand.Execute() → gọi ExecuteStart()
```

### 6.3. ViewModel chính — `RobotViewModel.cs`

**File:** `ViewModels/RobotViewModel.cs` (287 dòng)  
**Vai trò:** Bộ não trung tâm — liên kết giao diện với phần cứng. Xử lý:
1. **Data Binding** — 24+ thuộc tính hiển thị trên UI
2. **Commands** — Start/Stop/Home
3. **Edge Detection** — Phát hiện sườn SystemState
4. **Database Logging** — Thu thập và ghi dữ liệu

#### 6.3.1. Các trường nội bộ (Backing Fields)

```csharp
// ── Biến mong muốn (nhận từ UART) ──
private double _toaDoX, _toaDoY, _gocAlpha;
private double _gocTheta1, _gocTheta2, _gocTheta3;
private double _momen1, _momen2, _momen3;

// ── Biến thực tế (tính từ FK/IK) ──
private double _actualX, _actualY, _actualAlpha;
private double _actualTheta1, _actualTheta2, _actualTheta3;

// ── Biến sai số (tính toán) ──
private double _errorX, _errorY, _errorAlpha;
private double _errorTheta1, _errorTheta2, _errorTheta3;

// ── Biến trạng thái ──
private string _trangThaiHeThong = "Chưa kết nối";
private string _selectedTrajectoryId = "2";   // Mặc định = Hình Tròn
private int _systemState = 0;

// ── Biến ghi log Database ──
private int _currentRunId = -1;           // -1 = chưa có lần chạy nào đang diễn ra
private List<RunData> _currentRunData = new(); // Danh sách tạm trong RAM
private DateTime _startTime;              // Thời điểm bắt đầu (sườn lên)
private int _previousSystemState = 0;     // Trạng thái frame trước (cho Edge Detection)
```

#### 6.3.2. Cơ chế Edge Detection — Trái tim của hệ thống ghi dữ liệu

```csharp
public void UpdateData(double x, double y, double alpha,
                       double t1, double t2, double t3,
                       double m1, double m2, double m3,
                       int currentSystemState)
{
    // ═══ EDGE DETECTION ═══
    if (_previousSystemState != currentSystemState)
    {
        if (_previousSystemState == 0 && currentSystemState == 1)
        {
            // ▶▶▶ SƯỜN LÊN (0→1): Robot vừa bắt đầu chạy ◀◀◀
            _startTime      = DateTime.Now;                           // Ghi thời điểm bắt đầu
            _currentRunData = new List<RunData>(2048);                // Tạo danh sách RAM mới (pre-allocated 2048 slots)
            _currentRunId   = DatabaseManager.Instance.StartNewRun(GetTrajectoryName()); // Tạo dòng mới trong DB → nhận RunID
            RunStarted?.Invoke(this, EventArgs.Empty);                // Thông báo cho UI
        }
        else if (_previousSystemState == 1 && currentSystemState == 0)
        {
            // ⏹⏹⏹ SƯỜN XUỐNG (1→0): Robot vừa dừng ◀◀◀
            SaveAndEndCurrentRun("Đã dừng (Từ VĐK)");               // Ghi RAM → DB + Cập nhật EndTime
        }
    }
    _previousSystemState = currentSystemState;  // Lưu lại cho lần so sánh tiếp theo

    // ═══ CẬP NHẬT UI (Data Binding) ═══
    ToaDoX = x; ToaDoY = y; GocAlpha = alpha;
    GocTheta1 = t1; GocTheta2 = t2; GocTheta3 = t3;
    Momen1 = m1; Momen2 = m2; Momen3 = m3;

    // ═══ THU THẬP DỮ LIỆU VÀO RAM ═══
    if (_currentRunId < 0 || currentSystemState != 1) return;  // Không ghi nếu chưa chạy

    double timeOffset = (DateTime.Now - _startTime).TotalSeconds;
    _currentRunData.Add(new RunData
    {
        RunID      = _currentRunId,
        TimeOffset = timeOffset,
        Target_X   = x, Target_Y = y, Alpha = alpha,
        Theta1 = t1, Theta2 = t2, Theta3 = t3,
        Torque1 = m1, Torque2 = m2, Torque3 = m3
        // Actual/Error sẽ được bổ sung bởi UpdateDataFromRobotData()
    });
}
```

**Tại sao dùng Edge Detection thay vì nút bấm?**
> Vì trạng thái Start/Stop thực sự được xác nhận bởi **STM32** (qua byte SystemState), không phải bởi nút bấm. Nút bấm chỉ gửi **yêu cầu** — phần cứng mới **quyết định** có chạy hay không. Edge Detection đảm bảo ghi dữ liệu chính xác theo trạng thái thực tế của phần cứng.

#### 6.3.3. Bổ sung dữ liệu Actual/Error — `UpdateDataFromRobotData()`

```csharp
public void UpdateDataFromRobotData(RobotData data)
{
    // Gọi UpdateData() trước (Edge Detection + cập nhật mong muốn)
    UpdateData(data.PosX, data.PosY, data.Alpha, ...);

    // Bổ sung Actual/Error từ kết quả kinematics
    ActualX = data.ActualX; ActualY = data.ActualY; ...
    ErrorX = data.ErrorX; ErrorY = data.ErrorY; ...

    // Cập nhật ĐIỂM DỮ LIỆU CUỐI CÙNG trong _currentRunData
    // Vì UpdateData() chỉ ghi giá trị mong muốn, Actual/Error cần bổ sung sau
    if (_currentRunId >= 0 && _currentRunData.Count > 0)
    {
        var last = _currentRunData[_currentRunData.Count - 1];
        last.X = data.ActualX;       // Bổ sung XY thực tế
        last.Y = data.ActualY;
        last.ActualAlpha  = data.ActualAlpha;
        last.ActualTheta1 = data.ActualTheta1;
        // ... (tất cả 12 trường Actual + Error)
    }
}
```

#### 6.3.4. Lưu và kết thúc lần chạy

```csharp
private void SaveAndEndCurrentRun(string status)
{
    if (_currentRunId < 0) return;  // Không có lần chạy nào → bỏ qua

    try
    {
        // Ghi toàn bộ _currentRunData (RAM) vào SQLite (ổ cứng) trong 1 transaction
        DatabaseManager.Instance.EndRun(_currentRunId, status, _currentRunData);
    }
    finally
    {
        _currentRunData.Clear();  // Giải phóng RAM
        _currentRunId = -1;       // Reset trạng thái "không có lần chạy nào"
    }
}
```

#### 6.3.5. Bản đồ quỹ đạo

```csharp
private string GetTrajectoryName() => SelectedTrajectoryId switch
{
    "1" => "Hình Vuông",     "2" => "Hình Tròn",
    "3" => "Đường Thẳng",    "4" => "Hình Sao",
    "5" => "Hình Hạc",       "6" => "Vẽ Chữ A",
    "7" => "Vẽ Chữ B",       "8" => "Vẽ Chữ DUT",
    "9" => "Vẽ Chữ TDH",
    _   => $"Quỹ đạo #{SelectedTrajectoryId}"
};
```

### 6.4. `SharedRobotViewModel` — Singleton xuyên suốt ứng dụng

Trong `App.xaml.cs`, ViewModel được tạo **1 lần duy nhất** và chia sẻ giữa tất cả cửa sổ:

```csharp
public partial class App : Application
{
    // ViewModel duy nhất — static property, tạo 1 lần khi app khởi chạy
    public static RobotViewModel SharedRobotViewModel { get; } = new RobotViewModel();

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        DatabaseManager.Instance.InitializeDatabase();  // Tạo DB nếu chưa có
    }
}
```

**Tại sao phải dùng chung 1 ViewModel?** Vì Edge Detection dựa vào `_previousSystemState`. Nếu mỗi trang tạo ViewModel mới, `_previousSystemState` luôn = 0 → không phát hiện được sườn → ghi dữ liệu sai.

---

## CHƯƠNG 7: XÂY DỰNG TẦNG DỮ LIỆU

### 7.1. Cấu trúc cơ sở dữ liệu — `DbModels.cs`

#### Bảng `RunHistory` — Tổng quan mỗi lần chạy

| Cột | Kiểu | Ý nghĩa | Ví dụ |
|-----|------|---------|-------|
| `RunID` | INTEGER (tự tăng) | Số thứ tự lần chạy | 1, 2, 3... |
| `TrajName` | TEXT | Tên quỹ đạo | "Hình Vuông" |
| `StartTime` | TEXT (ISO 8601) | Thời điểm sườn lên | 2026-04-15T09:30:00.000 |
| `EndTime` | TEXT (ISO 8601) | Thời điểm sườn xuống | 2026-04-15T09:31:25.000 |
| `EndStatus` | TEXT | Trạng thái kết thúc | "Hoàn thành" / "Đã dừng (Từ VĐK)" / "Fault" |

**Thuộc tính tính toán (computed properties):**
```csharp
public string EndTimeDisplay =>
    EndTime == DateTime.MinValue ? "Đang chạy..." : EndTime.ToString("dd/MM/yyyy HH:mm:ss");

public string DurationDisplay
{
    get
    {
        if (EndTime == DateTime.MinValue) return "—";
        TimeSpan duration = EndTime - StartTime;
        return duration.TotalHours >= 1
            ? duration.ToString(@"hh\:mm\:ss")   // ≥ 1 giờ: hiện giờ:phút:giây
            : duration.ToString(@"mm\:ss");       // < 1 giờ: hiện phút:giây
    }
}
```

#### Bảng `RunData` — Từng mẫu dữ liệu chi tiết (24 cột)

| Nhóm | Cột | Nguồn |
|------|-----|-------|
| **Khóa** | DataID, RunID, TimeOffset | Tự sinh / PC |
| **XY mong muốn** | Target_X, Target_Y, Alpha | UART |
| **XY thực tế** | X, Y, ActualAlpha | FK |
| **Theta mong muốn** | Theta1, Theta2, Theta3 | UART |
| **Theta thực tế** | ActualTheta1, ActualTheta2, ActualTheta3 | IK |
| **Momen** | Torque1, Torque2, Torque3 | UART |
| **Sai số XY** | ErrorX, ErrorY, ErrorAlpha | Tính |
| **Sai số Theta** | Error1, Error2, Error3 | Tính |

### 7.2. Quản lý Database — `DatabaseManager.cs`

**File:** `Services/DatabaseManager.cs` (489 dòng)  
**Mẫu thiết kế:** Singleton  

#### Khởi tạo Database

```csharp
public void InitializeDatabase()
{
    using var connection = new SqliteConnection(_connectionString);
    connection.Open();

    // Bật Foreign Key (đảm bảo tính toàn vẹn dữ liệu)
    using (var pragma = connection.CreateCommand())
    {
        pragma.CommandText = "PRAGMA foreign_keys = ON;";
        pragma.ExecuteNonQuery();
    }

    // Tạo bảng RunHistory (nếu chưa có)
    // Tạo bảng RunData (nếu chưa có) — 24 cột

    // ▶ MIGRATION: Thêm cột mới cho DB cũ (không mất dữ liệu)
    AddColumnIfMissing(connection, "RunData", "ActualAlpha",  "REAL NOT NULL DEFAULT 0");
    AddColumnIfMissing(connection, "RunData", "ActualTheta1", "REAL NOT NULL DEFAULT 0");
    // ... (8 cột mới)
}
```

#### Bulk Insert — Ghi hàng nghìn điểm trong 1 transaction

```csharp
public void EndRun(int runId, string status, List<RunData> collectedData)
{
    // 1. Cập nhật EndTime + EndStatus trong RunHistory
    // 2. Nếu có data → Bulk Insert:
    
    using var transaction = connection.BeginTransaction();
    try
    {
        // Chuẩn bị câu INSERT 1 lần, đổi parameters nhiều lần
        using var insertCmd = connection.CreateCommand();
        insertCmd.CommandText = "INSERT INTO RunData (...24 cột...) VALUES (...24 params...)";
        
        // Khai báo 24 parameters
        insertCmd.Parameters.Add("@RunID", SqliteType.Integer);
        // ... (24 parameters)
        
        // Lặp qua từng điểm dữ liệu
        foreach (var dp in collectedData)
        {
            insertCmd.Parameters["@RunID"].Value = runId;
            insertCmd.Parameters["@TimeOffset"].Value = dp.TimeOffset;
            // ... (gán 24 giá trị)
            insertCmd.ExecuteNonQuery();  // Ghi 1 dòng
        }
        
        transaction.Commit();    // Ghi tất cả cùng lúc (atomic)
    }
    catch
    {
        transaction.Rollback();  // Lỗi → hoàn tác toàn bộ
        throw;
    }
}
```

**Tại sao dùng transaction?**
- Không dùng transaction: 1800 điểm = 1800 lần ghi ổ cứng → ~5-10 giây
- Dùng transaction: 1800 điểm = 1 lần ghi ổ cứng → ~50ms (**nhanh hơn ~100 lần**)

---

## CHƯƠNG 8: XÂY DỰNG GIAO DIỆN NGƯỜI DÙNG

### 8.1. Điểm khởi động — `App.xaml`

```xml
<Application x:Class="HMI_WPF.App"
             StartupUri="Login.xaml">   <!-- Mở Login.xaml đầu tiên -->
</Application>
```

**Luồng khởi động:**
```
App khởi chạy
    ├── App.xaml.cs: OnStartup()
    │   ├── Tạo SharedRobotViewModel (1 lần duy nhất)
    │   └── DatabaseManager.Instance.InitializeDatabase()
    │       → Tạo file robot_log.db (nếu chưa có)
    │       → Tạo/cập nhật 2 bảng
    └── Mở Login.xaml
```

### 8.2. Màn hình Đăng nhập — `Login.xaml.cs`

```csharp
private void btnLogin_Click(object sender, RoutedEventArgs e)
{
    if (txtUsername.Text == "admin" && txtPassword.Password == "123")
    {
        MainWindow home = new MainWindow();
        home.WindowStartupLocation = WindowStartupLocation.CenterScreen;
        home.Show();       // Mở trang chủ
        this.Close();      // Đóng màn hình đăng nhập
    }
    else
    {
        MessageBox.Show("Sai tài khoản hoặc mật khẩu!", "Lỗi");
    }
}
```

### 8.3. Trang chủ SCADA — `MainWindow.xaml.cs`

#### Constructor — Khởi tạo toàn bộ hệ thống

```csharp
public MainWindow()
{
    InitializeComponent();

    // 1. Gắn ViewModel Singleton vào DataContext
    _robotViewModel = App.SharedRobotViewModel;
    this.DataContext = _robotViewModel;

    // 2. Khởi tạo đồng hồ + đồ họa robot
    InitializeClock();
    InitRobotGraphics();

    // 3. Khởi tạo LED Timer (100ms = 10Hz)
    _ledTimer = new DispatcherTimer { Interval = TimeSpan.FromMilliseconds(100) };
    _ledTimer.Tick += LedTimer_Tick;
    _ledTimer.Start();

    // 4. Đăng ký nhận sự kiện UART
    UartManager.Instance.DataReceived += UartManager_DataReceived;

    // 5. TỰ ĐỘNG kết nối COM3 khi khởi chạy
    UartManager.Instance.Start("COM3", 115200);

    // 6. Giải phóng COM khi tắt ứng dụng
    Application.Current.Exit += Current_Exit;
}
```

#### Digital Twin — Khởi tạo đồ họa robot

```csharp
public void InitRobotGraphics()
{
    // 0. Vẽ lưới tọa độ (Grid) — nền cho Canvas
    for (double i = -0.2; i <= 0.81; i += 0.1)
    {
        // Đường lưới ngang + dọc + Nhãn trục X/Y
    }

    // 1. Tạo 3 cánh tay chủ động (màu DodgerBlue, dày 8px)
    for (int i = 0; i < 3; i++)
    {
        activeArms[i] = new Line
        {
            Stroke = DodgerBlue, StrokeThickness = 8,
            StrokeStartLineCap = PenLineCap.Round  // Bo tròn đầu mút
        };
        RobotCanvas.Children.Add(activeArms[i]);
    }

    // 2. Tạo 3 cánh tay bị động (màu Tomato, dày 4px)
    // 3. Tạo mâm tam giác (Polygon, màu LimeGreen bán trong suốt)
    // 4. Tạo 3 bệ động cơ (Ellipse 24×24px)
}
```

#### Digital Twin — Cập nhật vị trí robot mỗi frame

```csharp
public void UpdateDigitalTwin(double posX, double posY, double alphaRad,
                               double theta1Rad, double theta2Rad, double theta3Rad)
{
    Application.Current.Dispatcher.Invoke(() =>
    {
        movingPlatform.Points.Clear();  // Xóa tam giác cũ

        for (int i = 0; i < 3; i++)
        {
            // Bước 1: Tính tọa độ khớp khuỷu A_i
            // A_i = M_i + L1 * [cos(theta_i), sin(theta_i)]
            double ax = M[i].X + L1 * Math.Cos(thetas[i]);
            double ay = M[i].Y + L1 * Math.Sin(thetas[i]);

            // Bước 2: Tính tọa độ chốt mâm B_i
            // B_i = [posX, posY] + L3 * [cos(gamma_i + alpha), sin(gamma_i + alpha)]
            double bx = posX + L3 * Math.Cos(gamma[i] + alphaRad);
            double by = posY + L3 * Math.Sin(gamma[i] + alphaRad);

            // Bước 3: Gán vào Line trên Canvas
            activeArms[i].X1  = MapX(M[i].X); activeArms[i].Y1  = MapY(M[i].Y);  // Gốc → Khuỷu
            activeArms[i].X2  = MapX(ax);     activeArms[i].Y2  = MapY(ay);
            passiveArms[i].X1 = MapX(ax);     passiveArms[i].Y1 = MapY(ay);       // Khuỷu → Mâm
            passiveArms[i].X2 = MapX(bx);     passiveArms[i].Y2 = MapY(by);

            // Tích lũy đỉnh tam giác
            movingPlatform.Points.Add(new Point(MapX(bx), MapY(by)));
        }

        // Vẽ vết quỹ đạo
        polyTrace.Points.Add(new Point(MapX(posX), MapY(posY)));
    });
}
```

**Hệ tọa độ chuyển đổi:**
```
Toán (mét):  Gốc ở (0,0), trục Y hướng lên
Canvas (pixel): Gốc ở (0,0) góc trên-trái, trục Y hướng xuống

MapX(x) = OFFSET_X + x * SCALE = 170 + x * 850
MapY(y) = OFFSET_Y - y * SCALE = 680 - y * 850
                      ↑ DẤU TRỪ: đảo chiều trục Y
```

#### Hệ thống LED trạng thái

```csharp
private void LedTimer_Tick(object? sender, EventArgs e)  // Chạy mỗi 100ms
{
    // 1. LED Run/Stop theo SystemState
    if (_currentSystemState == 1)
    {
        ledRun.Fill  = Brushes.LimeGreen;  // Xanh lá
        ledStop.Fill = Brushes.DarkGray;
    }
    else
    {
        ledRun.Fill  = Brushes.DarkGray;
        ledStop.Fill = Brushes.Red;         // Đỏ
    }

    // 2. Kiểm tra mất kết nối
    double timeSinceLastFrame = (DateTime.Now - UartManager.Instance.LastReceivedTime).TotalMilliseconds;
    _isUartFault = timeSinceLastFrame > 500;  // > 500ms = mất kết nối

    // 3. LED Fault nhấp nháy vàng (5Hz = bật/tắt mỗi 100ms)
    if (_isUartFault)
    {
        ledFault.Fill = _isFaultLedYellow ? Brushes.DarkGray : Brushes.Yellow;
        _isFaultLedYellow = !_isFaultLedYellow;
    }
    else
    {
        ledFault.Fill = Brushes.DarkGray;
    }
}
```

#### Cơ chế điều hướng trang

Mỗi nút điều hướng (Trends, Parameters, History) tạo **cửa sổ mới** rồi **đóng cửa sổ cũ**:

```csharp
private void btnTrends_Click(object sender, RoutedEventArgs e)
{
    Trends newWin = new Trends();
    newWin.Left = this.Left;    // Giữ vị trí cửa sổ
    newWin.Top = this.Top;
    newWin.Show();
    this.Close();
}
```

**Tại sao tạo cửa sổ mới thay vì dùng Frame/Page?**  
Vì WPF Page/Frame có nhiều hạn chế về lifecycle management. Cách tạo Window mới đơn giản hơn, và kết nối UART không bị ảnh hưởng vì `UartManager` là Singleton tồn tại độc lập.

### 8.4. Đồ thị thời gian thực — `Trends.xaml.cs`

#### Cơ chế Scatter Rebuild (30 FPS)

```csharp
private void RenderTimer_Tick(object? sender, EventArgs e)  // Mỗi 33ms
{
    lock (UartManager.Instance.XList)  // Khóa danh sách để tránh xung đột
    {
        int cnt = UartManager.Instance.TimeList.Count;
        double[] time = cnt > 0 ? UartManager.Instance.TimeList.ToArray() : Array.Empty<double>();

        // ═══ ĐỒ THỊ THETA ═══
        RemoveScatter(plotTheta.Plot, ref thetaDesScatter);   // 1. Xóa Scatter cũ
        RemoveScatter(plotTheta.Plot, ref thetaActScatter);

        if (cnt > 0)
        {
            // 2. Chọn kênh dữ liệu theo ComboBox
            List<double> desList = _thetaChannel switch
            {
                0 => UartManager.Instance.Theta1List,     // Theta 1
                1 => UartManager.Instance.Theta2List,     // Theta 2
                _ => UartManager.Instance.Theta3List      // Theta 3
            };

            // 3. Tạo Scatter mới — đường Mong muốn (xanh nhạt)
            thetaDesScatter = plotTheta.Plot.Add.ScatterLine(time, desList.ToArray());
            thetaDesScatter.Color = Color.FromHex("#4FC3F7");
            thetaDesScatter.LegendText = "Mong muốn";

            // 4. Tạo Scatter — đường Thực tế (hồng nhạt)
            thetaActScatter = plotTheta.Plot.Add.ScatterLine(time, actList.ToArray());
            thetaActScatter.Color = Color.FromHex("#EF9A9A");
            thetaActScatter.LegendText = "Thực tế";
        }

        // ═══ ĐỒ THỊ MOMEN ═══ (tương tự, hỗ trợ "Tất cả" hiện 3 kênh cùng lúc)
        // ═══ ĐỒ THỊ SAI SỐ ═══ (5 kênh: eTheta1, eTheta2, eTheta3, eX, eY)
        // ═══ ĐỒ THỊ QUỸ ĐẠO XY ═══ (2 đường: mong muốn Cyan + thực tế Orange)
    }

    // AutoScale tất cả trục
    plotTheta.Plot.Axes.AutoScale();
    plotTorque.Plot.Axes.AutoScale();
    plotError.Plot.Axes.AutoScale();

    // Refresh tất cả 4 đồ thị
    plotTrajectory.Refresh();
    plotTheta.Refresh();
    plotTorque.Refresh();
    plotError.Refresh();
}
```

**Tại sao dùng Scatter Rebuild thay vì append?**
- ScottPlot 5 API thay đổi so với v4 — không hỗ trợ append nhanh vào Scatter hiện có
- Rebuild toàn bộ đảm bảo đồng bộ với danh sách dữ liệu trong UartManager
- Hiệu suất vẫn đạt 30 FPS nhờ `ToArray()` + ScatterLine (không có marker)

#### Xử lý khi Start mới — Xóa đồ thị cũ

```csharp
private void UartManager_TrajectoryStarted(object? sender, EventArgs e)
{
    Application.Current.Dispatcher.Invoke(() =>
    {
        // Xóa tất cả Scatter (dữ liệu đã được ClearData() trong UartManager)
        RemoveScatter(plotTheta.Plot, ref thetaDesScatter);
        RemoveScatter(plotTheta.Plot, ref thetaActScatter);
        // ... (xóa tất cả 8 scatter)
        
        // Refresh để hiện đồ thị trống
        plotTrajectory.Refresh();
        plotTheta.Refresh();
        plotTorque.Refresh();
        plotError.Refresh();
    });
}
```

### 8.5. Bảng thông số — `Parameters.xaml.cs`

```csharp
private void UartManager_DataReceived(object? sender, RobotDataEventArgs e)
{
    // Chuyển từ background thread sang UI thread
    Application.Current.Dispatcher.Invoke(() =>
    {
        txtPosX.Text    = e.Data.PosX.ToString("F3");     // 3 chữ số thập phân
        txtPosY.Text    = e.Data.PosY.ToString("F3");
        txtAlpha.Text   = e.Data.Alpha.ToString("F3");
        txtErrX.Text    = e.Data.Error1.ToString("F3");
        txtErrY.Text    = e.Data.Error2.ToString("F3");
        txtErrAlpha.Text = e.Data.Error3.ToString("F3");
        txtTheta1.Text  = e.Data.Theta1.ToString("F3");
        txtTheta2.Text  = e.Data.Theta2.ToString("F3");
        txtTheta3.Text  = e.Data.Theta3.ToString("F3");
        txtTorque1.Text = e.Data.Torque1.ToString("F3");
        txtTorque2.Text = e.Data.Torque2.ToString("F3");
        txtTorque3.Text = e.Data.Torque3.ToString("F3");
    });
}
```

**Tại sao cần `Dispatcher.Invoke()`?**
> Sự kiện `DataReceived` được phát từ background thread (luồng UART). Nhưng các TextBlock thuộc UI thread. WPF cấm truy cập UI từ thread khác → bắt buộc phải dùng `Dispatcher.Invoke()` để chuyển sang UI thread.

**Quản lý tài nguyên — Tránh Memory Leak:**
```csharp
private void Parameters_Unloaded(object sender, RoutedEventArgs e)
{
    // QUAN TRỌNG: Hủy đăng ký sự kiện khi trang bị đóng
    UartManager.Instance.DataReceived -= UartManager_DataReceived;
    // Nếu quên bước này → mỗi lần mở-đóng trang = thêm 1 subscriber
    // → Sau 10 lần, 10 subscriber cùng cập nhật UI → giật lag → crash
}
```

### 8.6. Lịch sử vận hành — `History.xaml.cs`

#### Tải dữ liệu từ DB

```csharp
private void LoadHistoryData()
{
    DateTime fromDate    = dpFromDate.SelectedDate ?? DateTime.Now.AddDays(-30);
    DateTime toDate      = dpToDate.SelectedDate ?? DateTime.Now;
    string selectedTraj  = GetSelectedTrajectoryName();   // "Tất cả" hoặc tên quỹ đạo

    List<RunHistory> data = DatabaseManager.Instance
        .GetFilteredRunHistory(fromDate, toDate, selectedTraj);

    HistoryGrid.ItemsSource = data;   // Gắn vào DataGrid → hiển thị tự động
}
```

#### Xuất CSV

```csharp
private void btnExport_Click(object sender, RoutedEventArgs e)
{
    // 1. Lấy dữ liệu đang hiển thị trên DataGrid
    var data = HistoryGrid.ItemsSource as List<RunHistory>;

    // 2. Hiện hộp thoại lưu file
    var saveDialog = new SaveFileDialog
    {
        FileName = $"LichSu_{DateTime.Now:yyyyMMdd_HHmmss}",
        DefaultExt = ".csv"
    };

    if (saveDialog.ShowDialog() == true)
    {
        var sb = new StringBuilder();
        sb.AppendLine("STT,Tên Quỹ Đạo,Thời Gian Bắt Đầu,...");  // Header
        foreach (var row in data)
            sb.AppendLine(string.Join(",", row.RunID, ...));        // Data rows

        // Ghi UTF-8 có BOM (để Excel hiển thị tiếng Việt đúng)
        File.WriteAllText(saveDialog.FileName, sb.ToString(), Encoding.UTF8);
    }
}
```

#### Nút Xóa thông minh

```csharp
private void btnDelete_Click(object sender, RoutedEventArgs e)
{
    string selectedTraj = GetSelectedTrajectoryName();

    if (selectedTraj == "Tất cả")
    {
        // Chế độ XÓA TOÀN BỘ — Xác nhận 2 lần
        if (MessageBox.Show("CẢNH BÁO! Xóa tất cả?", ...) == Yes)
            if (MessageBox.Show("Xác nhận lần cuối!", ...) == Yes)
                DatabaseManager.Instance.DeleteAllRuns();
    }
    else
    {
        // Chế độ XÓA THEO BỘ LỌC — Xác nhận 1 lần
        if (MessageBox.Show($"Xóa quỹ đạo {selectedTraj}?", ...) == Yes)
            DatabaseManager.Instance.DeleteFilteredRuns(fromDate, toDate, selectedTraj);
    }

    LoadHistoryData();  // Reload bảng sau khi xóa
}
```

---

## CHƯƠNG 9: CƠ CHẾ HOẠT ĐỘNG TỔNG THỂ

### 9.1. Luồng dữ liệu Uplink — Từ STM32 đến màn hình

```
STM32 gửi 77 byte qua USB-UART
        │
        ▼
UartManager.ReadThread()          ← Background Thread, chạy liên tục
        │ Đọc từng byte
        ▼
UartManager.ProcessByte()         ← Ghép byte thành khung 77 byte
        │ Khi đủ 77 byte + Header/Terminator hợp lệ
        ▼
UartManager.ProcessCompleteFrame()
        │
        ├── 1. Parse 9 double + 1 byte SystemState
        ├── 2. Gọi FK: Theta → ActualXYAlpha
        ├── 3. Gọi IK: XYAlpha → ActualTheta
        ├── 4. Tính sai số: Error = Desired − Actual
        ├── 5. Lưu vào 21 danh sách Singleton (lock)
        └── 6. Phát sự kiện DataReceived
                │
                ├──→ MainWindow.UartManager_DataReceived()
                │    ├── _robotViewModel.UpdateData()       ← Edge Detection + DB logging
                │    │   └── Nếu sườn lên → StartNewRun()
                │    │   └── Nếu sườn xuống → SaveAndEndCurrentRun()
                │    └── UpdateDigitalTwin()                ← Vẽ lại robot trên Canvas
                │
                ├──→ Parameters.UartManager_DataReceived()
                │    └── Cập nhật 12 TextBlock                ← Bảng thông số
                │
                └──→ (Trends không dùng DataReceived — dùng RenderTimer riêng)
```

### 9.2. Luồng dữ liệu Downlink — Từ người dùng đến STM32

```
Người dùng nhấn nút Start
        │
        ▼ (XAML Binding)
RobotViewModel.StartCommand.Execute()
        │
        ▼
RobotViewModel.ExecuteStart()
        │ Kiểm tra kết nối
        ▼
UartManager.SendControlSignal(trajId, start=true, stop=false, home=false)
        │
        ├── 1. Tạo 8 byte: [0x7E 0x7E] [trajId] [1 0 0] [0x03 0x03]
        ├── 2. Gửi xuống COM (lock)
        ├── 3. ClearData() — xóa 21 danh sách
        └── 4. Phát TrajectoryStarted → Trends xóa đồ thị cũ

MainWindow.btnStart_Click()
        └── polyTrace.Points.Clear()  — xóa vết vẽ Digital Twin
```

### 9.3. Luồng ghi dữ liệu vào Database

```
SystemState: 0→1 (SƯỜN LÊN)
        │
        ▼
RobotViewModel.UpdateData() phát hiện Edge
        ├── _startTime = DateTime.Now
        ├── _currentRunData = new List<RunData>(2048)    ← Pre-allocate RAM
        ├── _currentRunId = DatabaseManager.StartNewRun("Hình Tròn")
        │   └── INSERT INTO RunHistory → return RunID
        └── Phát RunStarted event
        
        ▼ (Robot đang chạy, ~30 frame/giây)
Mỗi frame UART đến:
        ├── _currentRunData.Add(new RunData{...})        ← Thêm 1 điểm vào RAM
        └── UpdateDataFromRobotData() bổ sung Actual/Error vào điểm cuối

SystemState: 1→0 (SƯỜN XUỐNG)
        │
        ▼
RobotViewModel.SaveAndEndCurrentRun("Đã dừng (Từ VĐK)")
        │
        ▼
DatabaseManager.EndRun(runId, status, _currentRunData)
        ├── UPDATE RunHistory SET EndTime, EndStatus
        ├── BEGIN TRANSACTION
        ├── Foreach dp in _currentRunData:
        │       INSERT INTO RunData (24 cột)
        ├── COMMIT                                        ← Ghi 1 lần duy nhất
        ├── _currentRunData.Clear()                       ← Giải phóng RAM
        └── _currentRunId = -1
```

### 9.4. Đồ thị Trends hoạt động độc lập

```
DispatcherTimer (33ms = 30 FPS)
        │
        ▼
RenderTimer_Tick()
        │
        ├── lock(UartManager.Instance.XList)
        │   ├── Copy danh sách → mảng double[] (snapshot)
        │   ├── Xóa Scatter cũ → Tạo Scatter mới → Gắn vào Plot
        │   └── unlock
        │
        ├── AutoScale tất cả trục
        └── Refresh 4 đồ thị
```

---

## CHƯƠNG 10: CÁC VẤN ĐỀ KỸ THUẬT VÀ CÁCH GIẢI QUYẾT

### 10.1. Cross-Thread Exception

**Vấn đề:** Sự kiện `DataReceived` được phát từ background thread (luồng UART), nhưng UI chỉ có thể thao tác từ UI thread.

**Giải pháp:** Sử dụng `Application.Current.Dispatcher.Invoke()` hoặc `BeginInvoke()` để marshal hành động từ background thread sang UI thread.

```csharp
// Invoke(): Đồng bộ — chờ UI thread xử lý xong mới tiếp tục
Application.Current.Dispatcher.Invoke(() => { /* cập nhật UI */ });

// BeginInvoke(): Bất đồng bộ — không chờ, UI thread xử lý khi rảnh
Application.Current.Dispatcher.BeginInvoke(() => { /* cập nhật UI */ });
```

### 10.2. Memory Leak do Event Subscription

**Vấn đề:** Mỗi lần mở trang mới, đăng ký `+=` vào sự kiện của Singleton. Nếu không hủy `-=` khi đóng trang, garbage collector không thể thu hồi bộ nhớ vì Singleton vẫn giữ reference.

**Giải pháp:** Luôn hủy đăng ký trong `Unloaded`:
```csharp
this.Unloaded += (sender, e) =>
{
    UartManager.Instance.DataReceived -= UartManager_DataReceived;
};
```

### 10.3. Race Condition khi đọc/ghi danh sách

**Vấn đề:** Background thread (UART) ghi vào danh sách, UI thread (Trends) đọc danh sách cùng lúc → `InvalidOperationException`.

**Giải pháp:** Sử dụng `lock()`:
```csharp
// Viết (trong UartManager):
lock (XList) { XList.Add(value); }

// Đọc (trong Trends):
lock (UartManager.Instance.XList) { data = XList.ToArray(); }
```

### 10.4. Hiệu suất ghi Database

**Vấn đề:** Ghi 30 lần/giây vào ổ cứng → giật lag nghiêm trọng.

**Giải pháp:** Pattern "Buffer & Flush":
- **Buffer (Runtime):** Thu thập dữ liệu vào `List<RunData>` (RAM) suốt thời gian chạy
- **Flush (Stop):** Ghi toàn bộ vào SQLite trong 1 transaction duy nhất khi robot dừng
- **Pre-allocate:** `new List<RunData>(2048)` — cấp phát trước 2048 slot, tránh resize nhiều lần

### 10.5. Tương thích ngược Database

**Vấn đề:** Khi thêm cột mới (ActualTheta, Error...) vào bảng RunData, DB cũ không có các cột này → crash.

**Giải pháp:** Migration an toàn với `AddColumnIfMissing()`:
```csharp
private static void AddColumnIfMissing(SqliteConnection conn, string table, 
    string column, string definition)
{
    try
    {
        conn.CreateCommand()
            .CommandText = $"ALTER TABLE {table} ADD COLUMN {column} {definition};"
            .ExecuteNonQuery();
    }
    catch { /* Cột đã tồn tại → bỏ qua lỗi */ }
}
```

### 10.6. Đồng bộ trạng thái khi chuyển trang

**Vấn đề:** Khi chuyển từ MainWindow sang Trends rồi quay lại, `_previousSystemState` bị reset → Edge Detection phát hiện sai.

**Giải pháp:** Sử dụng `SharedRobotViewModel` (static Singleton trong `App.xaml.cs`) — tất cả cửa sổ dùng chung 1 ViewModel → `_previousSystemState` không bao giờ bị reset trừ khi tắt ứng dụng.

### 10.7. Lỗi số trong Động học

**Vấn đề:** Hàm `acos()` trả về NaN khi đầu vào ngoài [-1, 1] do sai số làm tròn.

**Giải pháp:** Clamp đầu vào trước khi gọi `acos()`:
```csharp
cosArg = Math.Max(-1.0, Math.Min(1.0, cosArg));
```

---

## CHƯƠNG 11: TỔNG KẾT VÀ ĐÁNH GIÁ

### 11.1. Tổng hợp các file và số dòng code

| File | Số dòng | Vai trò |
|------|---------|---------|
| `App.xaml.cs` | 28 | Điểm khởi động, SharedViewModel, DB init |
| `Login.xaml.cs` | 64 | Đăng nhập |
| `MainWindow.xaml.cs` | 501 | Trang chủ SCADA, Digital Twin, LED |
| `Trends.xaml.cs` | 347 | 4 đồ thị thời gian thực |
| `Parameters.xaml.cs` | 140 | Bảng thông số |
| `History.xaml.cs` | 367 | Lịch sử, CSV, xóa dữ liệu |
| `UartManager.cs` | 352 | Giao tiếp UART Singleton |
| `RobotData.cs` | 140 | Mô hình dữ liệu 24+ thuộc tính |
| `RobotKinematics.cs` | 227 | FK + IK |
| `RobotViewModel.cs` | 287 | ViewModel chính, Edge Detection, DB |
| `ViewModelBase.cs` | 34 | INotifyPropertyChanged |
| `RelayCommand.cs` | 40 | ICommand wrapper |
| `DbModels.cs` | 92 | RunHistory + RunData |
| `DatabaseManager.cs` | 489 | SQLite CRUD, Bulk Insert, Migration |
| **Tổng cộng** | **~3,108** | **14 file C#** |

### 11.2. Tóm tắt các design patterns

| Pattern | Số lần sử dụng | Đánh giá |
|---------|----------------|----------|
| Singleton | 3 (UartManager, DatabaseManager, SharedViewModel) | Hiệu quả, tránh xung đột resource |
| MVVM | 1 (RobotViewModel + ViewModelBase) | Tách biệt tốt UI/logic |
| Observer/Event | 3 (DataReceived, TrajectoryStarted, RunStarted) | Linh hoạt, dễ mở rộng |
| Edge Detection | 1 (UpdateData) | Ghi dữ liệu chính xác theo phần cứng |
| Bulk Insert | 1 (EndRun) | Hiệu suất ~100x so với ghi từng dòng |
| Migration | 1 (AddColumnIfMissing) | Tương thích ngược an toàn |

### 11.3. Kết quả đạt được

| Tiêu chí | Mục tiêu | Kết quả |
|----------|----------|---------|
| Tần suất cập nhật UI | 30 FPS | ✅ Đạt (~30 FPS, không giật lag) |
| Giao tiếp UART | 77 byte @ 30Hz | ✅ Đạt (parse chính xác, thread-safe) |
| Digital Twin | Mô phỏng 3 cánh tay + mâm | ✅ Đạt (cập nhật thời gian thực) |
| Đồ thị | 4 biểu đồ, chọn kênh | ✅ Đạt (Scatter rebuild 30 FPS) |
| Database | Lưu 24 cột/frame | ✅ Đạt (Bulk Insert < 100ms) |
| Quỹ đạo | 9 loại | ✅ Đạt (Vuông, Tròn, Thẳng, Sao, Hạc, A, B, DUT, TDH) |
| LED trạng thái | Run/Stop/Fault | ✅ Đạt (Fault nhấp nháy 5Hz) |
| Xuất CSV | UTF-8 tiếng Việt | ✅ Đạt |

### 11.4. Hướng phát triển

| Hạng mục | Mô tả |
|----------|-------|
| Đăng nhập nâng cao | Thay hardcode bằng database, hỗ trợ nhiều tài khoản + phân quyền |
| Xuất báo cáo PDF | Tự động tạo báo cáo vận hành dạng PDF với đồ thị |
| Alarm System | Cảnh báo khi sai số vượt ngưỡng, log event |
| Remote Monitoring | Kết nối mạng LAN/Internet để giám sát từ xa |
| 3D Digital Twin | Nâng cấp từ Canvas 2D sang 3D (WPF 3D hoặc Unity) |
| Data Analytics | Phân tích tổng hợp dữ liệu lịch sử (xu hướng, thống kê) |

---

> **📝 Ghi chú:**  
> Báo cáo này trình bày chi tiết quá trình thực hiện dự án từ khâu phân tích yêu cầu, lựa chọn công nghệ, thiết kế kiến trúc, đến xây dựng từng module cụ thể. Mỗi đoạn code quan trọng đều được giải thích cơ chế hoạt động, luồng dữ liệu (biến đi đâu, về đâu), và lý do đưa ra quyết định thiết kế.
