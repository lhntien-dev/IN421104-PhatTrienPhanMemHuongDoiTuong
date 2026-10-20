
* **Mục tiêu:** Thực hiện **Nhiệm vụ 1 (TODO 1)** theo yêu cầu đề bài: Chỉ ra và phân tích chi tiết ít nhất 4 vi phạm SOLID trong lớp `ClubManager.cs` (chỉ rõ vị trí dòng, nguyên lý vi phạm, hậu quả kỹ thuật và giải pháp refactor).

---

## 🔍 TỔNG QUAN HIỆN TRẠNG
Lớp `ClubManager` là một **God Class** (Antipattern) ôm đồm quá nhiều trách nhiệm trái ngược:
1. Kiểm tra tính hợp lệ của dữ liệu đầu vào (Validation).
2. Lưu trữ và quản lý danh sách CLB, thành viên trong bộ nhớ.
3. Ghi log trực tiếp ra tệp tin trên đĩa cứng (`File.AppendAllText`).
4. Gửi thông báo trực tiếp qua Console và email.
5. Tổng hợp số liệu và in báo cáo thống kê.

---

## 🚫 CHI TIẾT CÁC VI PHẠM NGUYÊN LÝ SOLID

### 1. Vi phạm nguyên lý S (Single Responsibility Principle — Đơn trách nhiệm)
* **Vị trí vi phạm trong mã nguồn:**
  * Phương thức `CreateClub(name, description, presidentEmail)` (Dòng 35–64)
  * Phương thức `AddMemberToClub(clubId, memberName, email)` (Dòng 67–84)
  * Phương thức `CreateEvent(clubId, title, date)` (Dòng 88–99)
  * Phương thức `PrintClubReport()` (Dòng 103–113)
* **Mô tả vi phạm:**
  * Mỗi phương thức đều làm từ 3 đến 4 việc hoàn toàn khác nhau. Ví dụ: `CreateClub` vừa validate tên CLB, vừa tạo GUID và thêm vào `_clubs`, vừa mở file `club-log.txt` để ghi log, vừa duyệt điều kiện gửi email cho chủ nhiệm.
  * Tương tự, `CreateEvent` vừa tạo sự kiện vừa tự động duyệt vòng lặp `foreach` gửi email cho từng thành viên.
* **Hậu quả kỹ thuật:**
  * Lớp có quá nhiều lý do để thay đổi (Too many reasons to change). Bất kỳ thay đổi nào liên quan đến định dạng log, cách thức gửi email hay mẫu báo cáo đều buộc phải chỉnh sửa `ClubManager`.
  * Khó đọc, khó bảo trì và nguy cơ sinh lỗi chéo cao: sửa phần log có thể làm hỏng nghiệp vụ tạo CLB.
* **Giải pháp tái cấu trúc:**
  * Chuyển quy tắc hợp lệ tên và sức chứa về chính các thực thể Domain: `Club.Create()`, `Club.HasRoom()`.
  * Tách việc điều phối nghiệp vụ thành các dịch vụ độc lập: `ClubService`, `EventService`, `ReportService`.
  * Tách việc ghi log sang `IAppLogger` và việc gửi thông báo sang `INotificationService`.

---

### 2. Vi phạm nguyên lý O (Open/Closed Principle — Đóng / Mở)
* **Vị trí vi phạm trong mã nguồn:**
  * Chuỗi `if/else` phân loại gửi thông báo trong `CreateClub` (Dòng 52–63):
    ```csharp
    if (presidentEmail.Contains("@gmail.com"))
        Console.WriteLine($"Sending Gmail notification to {presidentEmail}");
    else if (presidentEmail.Contains("@yahoo.com"))
        Console.WriteLine($"Sending Yahoo notification to {presidentEmail}");
    else
        Console.WriteLine($"Sending generic email to {presidentEmail}");
    ```
  * In báo cáo cứng trong `PrintClubReport` (Dòng 103–113): Gắn chặt vào `Console.WriteLine`.
* **Mô tả vi phạm:**
  * Mã nguồn bị "đóng với mở rộng" và "mở với sửa đổi". Khi muốn bổ sung kênh thông báo mới (SMS, Slack, Zalo) hoặc định dạng báo cáo mới (CSV, JSON, PDF), ta bắt buộc phải mở file `ClubManager.cs` ra sửa trực tiếp.
* **Hậu quả kỹ thuật:**
  * Vi phạm đóng gói, tăng nguy cơ gây lỗi hồi quy (regression bugs) cho các tính năng đang chạy ổn định.
* **Giải pháp tái cấu trúc:**
  * **Áp dụng Factory Method Pattern:** Định nghĩa `INotificationService` và `INotificationServiceFactory` với switch-case tập trung trên enum `NotificationChannel`.
  * **Áp dụng Strategy Pattern:** Định nghĩa `IReportStrategy` với method `Render(stats)`. Tách thành `ConsoleReportStrategy` và `CsvReportStrategy`.

---

### 3. Vi phạm nguyên lý D (Dependency Inversion Principle — Đảo ngược phụ thuộc)
* **Vị trí vi phạm trong mã nguồn:**
  * Gọi trực tiếp API tệp tin: `File.AppendAllText(_logFilePath, ...)` (Dòng 48, 79, 94, 111)
  * Gọi trực tiếp Console: `Console.WriteLine(...)` (Dòng 54, 58, 62, 82, 83, 97, 108)
  * Khởi tạo trực tiếp đối tượng cụ thể bằng toán tử `new` bên trong lớp.
* **Mô tả vi phạm:**
  * Module cấp cao (`ClubManager`) bị phụ thuộc trực tiếp vào các chi tiết cấp thấp và phần cứng I/O của hệ điều hành (`File`, `Console`) thay vì phụ thuộc vào Abstraction (Interface).
* **Hậu quả kỹ thuật:**
  * **Không thể viết Unit Test cô lập:** Khi chạy test cho `ClubManager`, chương trình sẽ tự động ghi tệp `club-log.txt` ra đĩa cứng và in ra màn hình console, gây chậm, phụ thuộc quyền truy cập ổ đĩa và sinh file rác.
* **Giải pháp tái cấu trúc:**
  * Trừu tượng hóa việc ghi log qua interface `IAppLogger` (với các cài đặt `ConsoleLogger`, `FileLogger`).
  * Trừu tượng hóa kho lưu trữ qua `IClubRepository`, `IMemberRepository`, `IEventRepository`.
  * Áp dụng **Dependency Injection qua constructor**: `ClubService` chỉ nhận các interface từ bên ngoài, biến `Program.cs` thành **Composition Root** duy nhất.

---

### 4. Vi phạm nguyên lý I (Interface Segregation Principle — Phân tách giao diện)
* **Vị trí vi phạm trong mã nguồn:**
  * Toàn bộ lớp `ClubManager` không kế thừa bất kỳ interface nào và quản lý chung cả CLB, Thành viên lẫn Sự kiện.
* **Mô tả vi phạm:**
  * Nếu một module khác trong hệ thống EduHub (ví dụ: máy Kiosk điểm danh sinh viên tại hội trường) muốn sử dụng chức năng điểm danh, nó sẽ bị ép buộc phải phụ thuộc vào toàn bộ `ClubManager` cồng kềnh với các chức năng tạo CLB, in báo cáo mà nó không hề cần tới.
* **Hậu quả kỹ thuật:**
  * Độ phụ thuộc (Coupling) cao, khó tái sử dụng mã nguồn cho các nền tảng khác nhau (Web, Mobile, Console, Background Worker).
* **Giải pháp tái cấu trúc:**
  * Chia nhỏ thành các interface hẹp, chuyên biệt theo nghiệp vụ: `IClubRepository`, `IMemberRepository`, `IEventRepository`, `IClubService`, `IEventService`, `IReportService`.

---

## 🎯 KẾT LUẬN & ĐỊNH HƯỚNG REFACTOR
Để khắc phục hoàn toàn 4 vi phạm trên, giải pháp cần thực hiện gồm 4 nhiệm vụ cốt lõi:
1. **Tách dịch vụ & kho dữ liệu:** Phân rã `ClubManager` thành `ClubService`, `EventService`, `ReportService` và các Repository interface.
2. **Cài đặt 4 Design Patterns:**
   * **Factory:** `NotificationServiceFactory` chọn kênh thông báo Email, SMS, Slack.
   * **Decorator:** `LoggingClubRepository` bọc ngoài repository để ghi log trong suốt.
   * **Strategy:** `IReportStrategy` cho phép xuất báo cáo Console và CSV.
   * **Observer:** `IEventObserver` cho phép `EventService` tự động bắn thông báo khi tạo sự kiện mới.
3. **Hoàn thiện tầng Domain:** Đưa toàn bộ kiểm tra Invariants vào `Club.Create()`, `Club.HasRoom()`, `Event.Register()`, `Event.CheckIn()`.
4. **Viết Unit Test xUnit:** Hoàn thiện ít nhất 6 test AAA trong `DomainTests.cs` (kiểm tra ngoại lệ và kiểm tra giá trị biên).
