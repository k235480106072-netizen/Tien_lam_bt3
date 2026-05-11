#  BÀI TẬP VỀ NHÀ 03: HỆ QUẢN TRỊ CSDL - QUẢN LÝ CẦM ĐỒ

Thông tin sinh viên:

Họ và tên: Vi Trần Tiến

Mã SV: K235480106072

Lớp: 59KMT

Giảng viên hướng dẫn: Đỗ Duy Cốp

1. TỔNG QUAN BÀI TOÁN VÀ QUY TẮC NGHIỆP VỤ CỐT LÕI

Hệ thống CSDL phục vụ quản lý hợp đồng vay thế chấp tài sản tại tiệm cầm đồ với các đặc điểm:

Tính lãi kép có điều kiện: Lãi đơn áp dụng trước Deadline 1 (0.5%/ngày). Sau Deadline 1, lãi đơn nhập gốc và bắt đầu tính lãi kép theo ngày.

Toàn vẹn vòng đời tài sản: Trạng thái tài sản phụ thuộc vào trạng thái Hợp đồng và dòng tiền thanh toán (Audit Log).

Quản lý rủi ro: Chỉ cho phép khách rút một phần tài sản nếu giá trị định giá của các tài sản bị giữ lại vẫn đảm bảo cover được tổng dư nợ hiện tại.

2. NHIỆM VỤ 1: THIẾT KẾ CSDL (ERD & KHỞI TẠO BẢNG)


<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/fce6144d-63d6-41e1-931e-5a824e7c8579" />

Sơ đồ ERD

Phân tích thiết kế chuẩn 3NF:

Tách LogGiaoDich thành bảng riêng: Bắt buộc để đáp ứng "Sự kiện bổ sung (Audit Log)". Việc lưu dòng tiền giúp tính toán lại dư nợ tại bất kỳ thời điểm nào trong 
quá khứ mà không bị mất dấu vết, thay vì chỉ tạo 1 cột TongNo trong bảng HopDong và ghi đè liên tục.

Bảng TaiSan có khóa ngoại MaHD. Một hợp đồng (MaHD) có thể chứa nhiều tài sản (MaTS).


Code tạo cấu trúc & Dữ liệu mẫu (Sample Data Setup)

Khởi chạy đoạn code này đầu tiên để có môi trường Test cho các Event bên dưới.

```sql
CREATE DATABASE QuanLyCamDo_DB_ViTranTien;
GO
USE QuanLyCamDo_DB_ViTranTien;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/13a85703-e587-4240-b383-8cf2da145030" />
Tạo database QuanLyCamDo_DB_ViTranTien

```
CREATE TABLE KhachHang (
    MaKH INT IDENTITY(1,1) PRIMARY KEY,
    TenKH NVARCHAR(100) NOT NULL,
    SoDienThoai VARCHAR(15) UNIQUE NOT NULL
);

CREATE TABLE HopDong (
    MaHD INT IDENTITY(1,1) PRIMARY KEY,
    MaKH INT NOT NULL FOREIGN KEY REFERENCES KhachHang(MaKH),
    TienGoc DECIMAL(18,2) NOT NULL,
    NgayVay DATE NOT NULL DEFAULT GETDATE(),
    Deadline1 DATE NOT NULL,
    Deadline2 DATE NOT NULL,
    TrangThai NVARCHAR(50) DEFAULT N'Đang vay' 
);

CREATE TABLE TaiSan (
    MaTS INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT NOT NULL FOREIGN KEY REFERENCES HopDong(MaHD),
    TenTaiSan NVARCHAR(200) NOT NULL,
    GiaTriDinhGia DECIMAL(18,2) NOT NULL,
    TrangThai NVARCHAR(50) DEFAULT N'Đang cầm cố'
);

CREATE TABLE LogGiaoDich (
    MaLog INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT NOT NULL FOREIGN KEY REFERENCES HopDong(MaHD),
    NgayTra DATETIME DEFAULT GETDATE(),
    SoTienTra DECIMAL(18,2) NOT NULL,
    NguoiThuTien NVARCHAR(100),
    GhiChu NVARCHAR(255)
);
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/6c769489-c6e2-40a6-9cb4-8f5e8650de89" />

Tạo các bảng 

Chèn dữ liệu:

```sql
-- XÓA DỮ LIỆU CŨ NẾU CÓ (Để có thể chạy lại file này nhiều lần không bị lỗi)
-- Chú ý thứ tự xóa: Xóa bảng con trước, bảng cha sau.
DELETE FROM LogGiaoDich;
DELETE FROM TaiSan;
DELETE FROM HopDong;
DELETE FROM KhachHang;

-- Reset lại ID tự tăng về 1
DBCC CHECKIDENT ('KhachHang', RESEED, 0);
DBCC CHECKIDENT ('HopDong', RESEED, 0);
DBCC CHECKIDENT ('TaiSan', RESEED, 0);
DBCC CHECKIDENT ('LogGiaoDich', RESEED, 0);
GO

-- ======================================================
-- 1. THÊM DỮ LIỆU KHÁCH HÀNG
-- ======================================================
INSERT INTO KhachHang (TenKH, SoDienThoai)
VALUES
    (N'Nguyễn Văn A', '0901234567'),
    (N'Trần Thị B', '0987654321'),
    (N'Lê Hoàng C', '0911222333');
GO

-- ======================================================
-- 2. THÊM DỮ LIỆU HỢP ĐỒNG (Các kịch bản khác nhau)
-- ======================================================
-- Kịch bản 1 (Mã KH 1): Vay 10 ngày trước -> ĐANG VAY, tính lãi đơn.
-- Deadline 1: 30 ngày (còn 20 ngày nữa) / Deadline 2: 60 ngày.
INSERT INTO HopDong (MaKH, TienGoc, NgayVay, Deadline1, Deadline2, TrangThai)
VALUES (
    1, 15000000, 
    DATEADD(DAY, -10, GETDATE()), 
    DATEADD(DAY, 20, GETDATE()), 
    DATEADD(DAY, 50, GETDATE()), 
    N'Đang vay'
);

-- Kịch bản 2 (Mã KH 2): Vay 40 ngày trước -> QUÁ HẠN (Nợ xấu), đang tính lãi kép.
-- Quá Deadline 1 (10 ngày trước), nhưng chưa tới Deadline 2 (còn 20 ngày nữa).
INSERT INTO HopDong (MaKH, TienGoc, NgayVay, Deadline1, Deadline2, TrangThai)
VALUES (
    2, 50000000, 
    DATEADD(DAY, -40, GETDATE()), 
    DATEADD(DAY, -10, GETDATE()), 
    DATEADD(DAY, 20, GETDATE()), 
    N'Quá hạn (nợ xấu)'
);

-- Kịch bản 3 (Mã KH 3): Vay 80 ngày trước -> ĐÃ THANH LÝ.
-- Quá cả Deadline 1 và Deadline 2.
INSERT INTO HopDong (MaKH, TienGoc, NgayVay, Deadline1, Deadline2, TrangThai)
VALUES (
    3, 30000000, 
    DATEADD(DAY, -80, GETDATE()), 
    DATEADD(DAY, -50, GETDATE()), 
    DATEADD(DAY, -20, GETDATE()), 
    N'Đã thanh lý tài sản'
);
GO

-- ======================================================
-- 3. THÊM DỮ LIỆU TÀI SẢN
-- ======================================================
-- Tài sản của Hợp đồng 1 (HD đang vay nên tài sản Đang cầm cố)
INSERT INTO TaiSan (MaHD, TenTaiSan, GiaTriDinhGia, TrangThai)
VALUES
    (1, N'Laptop Macbook Pro M1', 18000000, N'Đang cầm cố'),
    (1, N'Apple Watch Series 7', 5000000, N'Đang cầm cố');

-- Tài sản của Hợp đồng 2 (HD nợ xấu, tài sản Đang cầm cố chờ thanh lý)
INSERT INTO TaiSan (MaHD, TenTaiSan, GiaTriDinhGia, TrangThai)
VALUES
    (2, N'Xe máy Honda SH 150i', 65000000, N'Đang cầm cố');

-- Tài sản của Hợp đồng 3 (HD đã thanh lý, tài sản cũng tự động đổi trạng thái Đã bán)
INSERT INTO TaiSan (MaHD, TenTaiSan, GiaTriDinhGia, TrangThai)
VALUES
    (3, N'Điện thoại iPhone 13 Pro Max', 20000000, N'Đã bán thanh lý'),
    (3, N'Máy ảnh Sony A7III', 25000000, N'Đã bán thanh lý');
GO

-- ======================================================
-- 4. THÊM LỊCH SỬ GIAO DỊCH (AUDIT LOG)
-- ======================================================
-- Khách A (Hợp đồng 1) đã ghé tiệm trả trước 2 triệu vào 5 ngày trước
INSERT INTO LogGiaoDich (MaHD, NgayTra, SoTienTra, NguoiThuTien, GhiChu)
VALUES
    (1, DATEADD(DAY, -5, GETDATE()), 2000000, N'Admin', N'Khách đóng tiền trả bớt nợ gốc đợt 1');

-- Khách B (Hợp đồng 2) từng đóng 5 triệu vào 20 ngày trước, nhưng sau đó bặt vô âm tín
INSERT INTO LogGiaoDich (MaHD, NgayTra, SoTienTra, NguoiThuTien, GhiChu)
VALUES
    (2, DATEADD(DAY, -20, GETDATE()), 5000000, N'Nhân viên 01', N'Thu tiền nợ đợt 1');
GO

-- ======================================================
-- XUẤT THỬ DỮ LIỆU KIỂM TRA
-- ======================================================
PRINT N'--- DANH SÁCH KHÁCH HÀNG ---'
SELECT * FROM KhachHang;

PRINT N'--- DANH SÁCH HỢP ĐỒNG ---'
SELECT * FROM HopDong;

PRINT N'--- DANH SÁCH TÀI SẢN ---'
SELECT * FROM TaiSan;

PRINT N'--- LỊCH SỬ THANH TOÁN (AUDIT LOG) ---'
SELECT * FROM LogGiaoDich;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/a3467ff5-d9fa-4a9f-a7ba-0594d64af457" />

Chèn dữ liệu vào các bảng 


3. NHIỆM VỤ 2: CÀI ĐẶT CÁC SỰ KIỆN (EVENTS)

Event 1: Đăng ký hợp đồng mới

Phân tích logic:

Sử dụng Stored Procedure (SP) để đóng gói giao dịch mở hợp đồng.

Thay vì yêu cầu người dùng nhập Deadline, SP chỉ nhận Tiền Gốc và Ngày Vay, sau đó sử dụng hàm DATEADD để tự động phát sinh Deadline1 (+30 ngày) và Deadline2 (+60 ngày).

Sử dụng SCOPE_IDENTITY() để trả về mã Hợp đồng vừa tạo, giúp ứng dụng (Front-end) lấy được ID để insert tiếp các tài sản thế chấp.

Code cài đặt:

```sql
CREATE PROCEDURE SP_DangKyHopDong
    @MaKH INT,
    @TienGoc DECIMAL(18,2),
    @NgayVay DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    IF @NgayVay IS NULL SET @NgayVay = GETDATE();

    DECLARE @Deadline1 DATE = DATEADD(DAY, 30, @NgayVay);
    DECLARE @Deadline2 DATE = DATEADD(DAY, 60, @NgayVay);

    INSERT INTO HopDong (MaKH, TienGoc, NgayVay, Deadline1, Deadline2, TrangThai)
    VALUES (@MaKH, @TienGoc, @NgayVay, @Deadline1, @Deadline2, N'Đang vay');
    
    SELECT SCOPE_IDENTITY() AS MaHopDongMoi, N'Tạo hợp đồng thành công' as Message;
END;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/b49c1e2f-a40a-41a1-919a-01a84f5e1dc5" />
Tạo Đăng ký hợp đồng mới


Code thử nghiệm Event 1:

```sql
-- Tạo Hợp đồng 1: Vay 10 triệu, ngày vay 01/01/2024
EXEC SP_DangKyHopDong @MaKH = 1, @TienGoc = 10000000, @NgayVay = '2024-01-01';
-- Thêm tài sản cho HD 1
INSERT INTO TaiSan (MaHD, TenTaiSan, GiaTriDinhGia) VALUES 
(1, N'Laptop Macbook M2', 15000000), (1, N'iPhone 14', 12000000);

-- Tạo Hợp đồng 2: Vay 50 triệu, ngày vay 01/01/2024 (Để test nợ xấu)
EXEC SP_DangKyHopDong @MaKH = 2, @TienGoc = 50000000, @NgayVay = '2024-01-01';
INSERT INTO TaiSan (MaHD, TenTaiSan, GiaTriDinhGia) VALUES 
(2, N'Xe máy SH', 60000000);

SELECT * FROM HopDong; -- Kiểm tra Deadline đã tự sinh đúng chưa (+30 và +60 ngày)
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/694ec98e-8dd9-4fd5-85ff-afd33f4e2fc5" />


Event 2: Thuật toán tính toán công nợ (Lãi đơn & Lãi kép)

Phân tích logic toán học:

Lãi suất ($r$): 5.000đ / 1.000.000đ / ngày $\rightarrow r = 0.005$ (0.5%/ngày).

Tiền đã trả ($P_{paid}$): Dùng SUM(SoTienTra) từ LogGiaoDich tính đến ngày cần xem xét.

Kịch bản 1 ($Ngày \le Deadline1$): Áp dụng công thức lãi đơn.
$Tổng Tiền = Gốc + (Gốc \times r \times SốNgày)$

Kịch bản 2 ($Ngày > Deadline1$): Lãi kép.
Bước A: Chốt sổ số tiền tại Deadline 1: $Gốc\_Mới = Gốc + (Gốc \times r \times \Delta t_{Deadline1})$
Bước B: Tính lãi kép từ Deadline 1 đến hiện tại: $Tổng Tiền = Gốc\_Mới \times (1 + r)^{SốNgàyQuáHạn}$

Dư nợ cuối cùng: $Dư Nợ = Tổng Tiền - P_{paid}$. (Nếu < 0 thì gán bằng 0).

Code cài đặt (Function):

```sql
CREATE FUNCTION fn_TinhDuNo
(
    @MaHD INT,
    @TargetDate DATE
)
RETURNS DECIMAL(18,2)
AS
BEGIN
    DECLARE @TienGoc DECIMAL(18,2), @Deadline1 DATE, @NgayVay DATE;
    DECLARE @LaiSuat FLOAT = 0.005; 
    DECLARE @TongTien DECIMAL(18,2) = 0, @DaTra DECIMAL(18,2) = 0;

    SELECT @TienGoc = TienGoc, @Deadline1 = Deadline1, @NgayVay = NgayVay
    FROM HopDong WHERE MaHD = @MaHD;

    -- Lấy tổng đã trả (nếu có)
    SELECT @DaTra = ISNULL(SUM(SoTienTra), 0) 
    FROM LogGiaoDich WHERE MaHD = @MaHD AND CAST(NgayTra AS DATE) <= @TargetDate;

    IF @TargetDate <= @Deadline1
    BEGIN
        DECLARE @SoNgay INT = DATEDIFF(DAY, @NgayVay, @TargetDate);
        IF @SoNgay < 0 SET @SoNgay = 0;
        SET @TongTien = @TienGoc + (@TienGoc * @LaiSuat * @SoNgay);
    END
    ELSE
    BEGIN
        -- Chốt gốc và lãi đơn tại Deadline 1
        DECLARE @NgayLaiDon INT = DATEDIFF(DAY, @NgayVay, @Deadline1);
        DECLARE @GocVaLaiDon DECIMAL(18,2) = @TienGoc + (@TienGoc * @LaiSuat * @NgayLaiDon);
        
        -- Tính lãi kép cho số ngày vượt
        DECLARE @NgayLaiKep INT = DATEDIFF(DAY, @Deadline1, @TargetDate);
        SET @TongTien = @GocVaLaiDon * POWER((1.0 + @LaiSuat), @NgayLaiKep);
    END

    DECLARE @DuNo DECIMAL(18,2) = @TongTien - @DaTra;
    IF @DuNo < 0 SET @DuNo = 0;
    RETURN @DuNo;
END;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/d22669c1-3f46-4e03-8757-c9ef2b7f884e" />

Code thử nghiệm Event 2:

-- Test 1: Tính nợ của HD 1 sau 20 ngày (Lãi đơn). Gốc 10tr.
-- Kỳ vọng: 10,000,000 + (10,000,000 * 0.005 * 20) = 11,000,000
SELECT dbo.fn_TinhDuNo(1, '2024-01-21') AS No_Sau_20_Ngay_LaiDon;

<img width="1912" height="1078" alt="image" src="https://github.com/user-attachments/assets/9b7b0001-ac5e-492b-a7bc-4d83c588c39b" />

-- Test 2: Tính nợ của HD 1 sau 35 ngày (Lãi kép 5 ngày). 
-- Gốc tại D1 (30 ngày) = 11,500,000. 
-- Kép 5 ngày = 11,500,000 * (1.005)^5 = ~ 11,790,381.56
SELECT dbo.fn_TinhDuNo(1, '2024-02-05') AS No_Sau_35_Ngay_LaiKep;

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/883a8aa9-0894-4ec0-8311-44c75f48e5a8" />


Event 3: Xử lý thanh toán & Gợi ý trả đồ

Phân tích logic:

Cập nhật tiền vào LogGiaoDich. Gọi lại fn_TinhDuNo để xem dư nợ hiện tại.

Nếu $Dư Nợ = 0$: Trả hết đồ (Cập nhật trạng thái TaiSan thành Đã trả khách).

Nếu $Dư Nợ > 0$: Giải bài toán "Trả đồ an toàn".
Gọi $V_{total}$ là tổng giá trị tài sản đang giữ. $D$ là dư nợ. Khách muốn lấy món đồ có giá trị $v_i$.
Điều kiện để tiệm an toàn là: $V_{total} - v_i \ge D$.
Suy ra: $v_i \le V_{total} - D$.
Do đó, hệ thống sẽ gợi ý (SELECT) ra các tài sản có GiaTriDinhGia <= (Tổng giá trị giữ lại - Dư nợ).

Code cài đặt:

```sql
CREATE PROCEDURE SP_ThanhToan
    @MaHD INT,
    @SoTienTra DECIMAL(18,2)
AS
BEGIN
    DECLARE @DuNo DECIMAL(18,2), @TongGiaTriTaiSan DECIMAL(18,2), @PhanDuThua DECIMAL(18,2);

    IF (SELECT TrangThai FROM HopDong WHERE MaHD = @MaHD) = N'Đã thanh lý tài sản'
    BEGIN
        PRINT N'Hợp đồng đã bị thanh lý. Hủy giao dịch.'; RETURN;
    END

    -- Ghi Log
    INSERT INTO LogGiaoDich (MaHD, SoTienTra, GhiChu) VALUES (@MaHD, @SoTienTra, N'Thanh toán nợ');

    -- Tính dư nợ
    SET @DuNo = dbo.fn_TinhDuNo(@MaHD, GETDATE());

    IF @DuNo = 0
    BEGIN
        UPDATE HopDong SET TrangThai = N'Đã thanh toán' WHERE MaHD = @MaHD;
        UPDATE TaiSan SET TrangThai = N'Đã trả khách' WHERE MaHD = @MaHD AND TrangThai = N'Đang cầm cố';
        PRINT N'Tất toán thành công. Đã trả mọi tài sản.';
    END
    ELSE
    BEGIN
        PRINT N'Dư nợ còn lại: ' + CAST(@DuNo AS NVARCHAR(50));
        
        SELECT @TongGiaTriTaiSan = ISNULL(SUM(GiaTriDinhGia), 0)
        FROM TaiSan WHERE MaHD = @MaHD AND TrangThai = N'Đang cầm cố';

        SET @PhanDuThua = @TongGiaTriTaiSan - @DuNo;
        
        IF @PhanDuThua > 0
        BEGIN
            PRINT N'>> DANH SÁCH TÀI SẢN BẠN CÓ THỂ LẤY VỀ:';
            SELECT MaTS, TenTaiSan, GiaTriDinhGia 
            FROM TaiSan 
            WHERE MaHD = @MaHD AND TrangThai = N'Đang cầm cố' AND GiaTriDinhGia <= @PhanDuThua;
        END
        ELSE
            PRINT N'>> Chưa đủ điều kiện an toàn để lấy bớt tài sản về.';
    END
END;
GO
```
<img width="1905" height="1078" alt="image" src="https://github.com/user-attachments/assets/ab367069-fdba-4f90-8078-3e4bbffb63e5" />
Tạo SP_ThanhToan


Code thử nghiệm Event 3:

-- Dư nợ HD 1 hiện tại (giả sử gọi lúc này) là khoảng ~11tr. 
-- Tài sản đang giữ: Macbook (15tr) + iPhone (12tr) = 27tr. 
-- Trả 5 triệu vào hệ thống.
EXEC SP_ThanhToan @MaHD = 1, @SoTienTra = 5000000;
-- Kết quả in ra sẽ là phần dư thừa đủ để lấy chiếc iPhone (12tr) về.

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/936450f0-1530-432e-848c-c368fc6e4b08" />


Event 4: Truy vấn danh sách nợ xấu

Phân tích logic:

Sử dụng VIEW để đóng gói truy vấn phức tạp. Lọc ra các hợp đồng có GETDATE() > Deadline1 và trạng thái chưa tất toán.

Kết hợp hàm fn_TinhDuNo để dự đoán nợ hiện tại và nợ sau 30 ngày (do đang chịu lãi kép, con số nhảy rất nhanh, giúp nhân viên đánh giá mức độ khẩn cấp).

Code cài đặt:

```sql
CREATE VIEW VW_DanhSachNoXau AS
SELECT 
    kh.TenKH, kh.SoDienThoai, 
    hd.MaHD, hd.TienGoc, hd.Deadline1,
    DATEDIFF(DAY, hd.Deadline1, GETDATE()) AS SoNgayQuaHan_D1,
    dbo.fn_TinhDuNo(hd.MaHD, GETDATE()) AS DuNoHienTai,
    dbo.fn_TinhDuNo(hd.MaHD, DATEADD(DAY, 30, GETDATE())) AS DuNoDuBao_Sau30Ngay
FROM HopDong hd
JOIN KhachHang kh ON hd.MaKH = kh.MaKH
WHERE GETDATE() > hd.Deadline1 
  AND hd.TrangThai IN (N'Đang vay', N'Quá hạn (nợ xấu)')
  AND dbo.fn_TinhDuNo(hd.MaHD, GETDATE()) > 0;
GO
```

Code thử nghiệm Event 4:

-- Giả lập Hợp đồng 2 (vay ngày 01/01) đã quá hạn so với ngày hiện tại
SELECT * FROM VW_DanhSachNoXau;

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/19839b75-da4d-4742-a471-dd06034e8fb8" />


Event 5: Quản lý thanh lý tài sản (Triggers vs Jobs)

Phân tích kỹ thuật (Rất quan trọng): * Trong SQL Server, Trigger chỉ kích hoạt khi có lệnh thao tác dữ liệu (DML - Insert/Update/Delete). Trigger KHÔNG THỂ tự động chạy theo lịch trình thời gian (Ví dụ: tự đổi trạng thái lúc 00:00:01 do vượt Deadline).

Do đó, để đáp ứng 2 yêu cầu đầu tiên của Event 5 một cách đúng thực tế, ta cần một Stored Procedure chạy quét định kỳ (được trigger bởi SQL Server Agent Job).

Với yêu cầu thứ 3 ("Đổi trạng thái tài sản thành Đã bán khi Hợp đồng bị đổi thành Đã thanh lý"), ta sử dụng Trigger AFTER UPDATE vì đây là sự kiện phụ thuộc dữ liệu.

Code cài đặt (Job Procedure & Update Trigger):
```sql
-- YÊU CẦU 1 & 2: Procedure xử lý quá hạn do thời gian (Chạy Job Daily)
CREATE PROCEDURE SP_Job_ChuyenTrangThaiQuaHan
AS
BEGIN
    -- 1. Vượt Deadline 1: Chuyển Hợp Đồng -> Quá hạn
    UPDATE HopDong SET TrangThai = N'Quá hạn (nợ xấu)'
    WHERE TrangThai = N'Đang vay' AND GETDATE() > Deadline1;

    -- 2. Vượt Deadline 2: Chuyển Tài Sản -> Sẵn sàng thanh lý
    UPDATE TaiSan SET TrangThai = N'Sẵn sàng thanh lý'
    WHERE TrangThai = N'Đang cầm cố'
      AND MaHD IN (SELECT MaHD FROM HopDong WHERE GETDATE() > Deadline2);
END;
GO

-- YÊU CẦU 3: Trigger cascade trạng thái khi bị Thanh lý
CREATE TRIGGER trg_HopDong_ThanhLyTaiSan
ON HopDong
AFTER UPDATE
AS
BEGIN
    -- Kiểm tra xem có phải update vào cột TrangThai và giá trị mới là Đã thanh lý không
    IF UPDATE(TrangThai)
    BEGIN
        UPDATE TaiSan SET TrangThai = N'Đã bán thanh lý'
        FROM TaiSan ts
        INNER JOIN inserted i ON ts.MaHD = i.MaHD
        WHERE i.TrangThai = N'Đã thanh lý tài sản'
          AND ts.TrangThai IN (N'Đang cầm cố', N'Sẵn sàng thanh lý');
    END
END;
GO
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/ef20b2e2-cc30-4cd4-8f5b-d903a5bfefe3" />
Tạo các trigger

Code thử nghiệm Event 5:

```sql
-- Chạy Job giả lập việc quét dữ liệu hàng ngày
EXEC SP_Job_ChuyenTrangThaiQuaHan;

-- Update trực tiếp Hợp đồng 2 sang trạng thái 'Đã thanh lý tài sản' để bẫy Trigger
UPDATE HopDong SET TrangThai = N'Đã thanh lý tài sản' WHERE MaHD = 2;

-- Kiểm tra bảng Tài sản của HD 2: Xe máy SH phải chuyển thành 'Đã bán thanh lý'
SELECT * FROM TaiSan WHERE MaHD = 2;
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/741c537f-8b0c-40d4-a4db-13fdcc5d079a" />

Sự kiện bổ sung: Gia hạn hợp đồng

Phân tích logic:

Gia hạn bản chất là ngăn không cho tính lãi kép. Khách hàng phải thanh toán toàn bộ số tiền lãi đã phát sinh từ đầu đến giờ. (Tiền nợ lúc này chỉ còn lại đúng bằng Tiền Gốc).

Reset lại Deadline1 (+30 ngày kể từ lúc gia hạn) và Deadline2 (+60 ngày). Cập nhật trạng thái về Đang vay.

Code cài đặt & Thử nghiệm:
```sql
CREATE PROCEDURE SP_GiaHanHopDong
    @MaHD INT
AS
BEGIN
    DECLARE @DuNo DECIMAL(18,2), @TienGoc DECIMAL(18,2), @LaiPhaiTra DECIMAL(18,2);
    
    SET @DuNo = dbo.fn_TinhDuNo(@MaHD, GETDATE());
    SELECT @TienGoc = TienGoc FROM HopDong WHERE MaHD = @MaHD;
    
    SET @LaiPhaiTra = @DuNo - @TienGoc;

    IF @LaiPhaiTra > 0
    BEGIN
        -- Thu toàn bộ tiền lãi
        INSERT INTO LogGiaoDich (MaHD, SoTienTra, GhiChu) 
        VALUES (@MaHD, @LaiPhaiTra, N'Thu lãi tồn đọng để gia hạn');
    END

    -- Reset kỳ hạn
    UPDATE HopDong
    SET Deadline1 = DATEADD(DAY, 30, GETDATE()),
        Deadline2 = DATEADD(DAY, 60, GETDATE()),
        TrangThai = N'Đang vay'
    WHERE MaHD = @MaHD;

    PRINT N'Đã gia hạn thành công. Thu tiền lãi: ' + CAST(@LaiPhaiTra AS NVARCHAR(50));
END;
GO
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/845d0c94-335c-4e24-9820-dc4176da1db5" />
Tạo SP_GiaHanHopDong
-- TEST:
```sql
-- EXEC SP_GiaHanHopDong @MaHD = 1;
-- SELECT * FROM HopDong WHERE MaHD = 1;
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/96028435-9d40-4279-98a9-5ef2c09b5c66" />
Test SP_GiaHanHopDong


Sự kiện bổ sung 2: Lịch sử hợp đồng (Audit Log) & Truy xuất dòng tiền

Phân tích logic:

Việc ghi đè (Overwrite) lên một cột như SoTienDaTra hoặc DuNoHienTai trong bảng Hợp đồng là tối kỵ trong phần mềm tài chính. CSDL bắt buộc phải có bảng Audit Log (LogGiaoDich) hoạt động theo nguyên tắc Append-Only (Chỉ thêm mới).

Lợi ích:

Ghi nhận chính xác khách trả tiền lắt nhắt thành nhiều đợt (Ngày trả, số tiền, người thu).

Tránh việc nhân viên gian lận hoặc sửa đổi số liệu.

Hàm fn_TinhDuNo được xây dựng dựa trên việc lấy Sum() từ bảng này, đảm bảo tính nhất quán tuyệt đối của thuật toán.

Code truy xuất Audit Log (Thử nghiệm):
Để theo dõi lịch sử dòng tiền của một hợp đồng cụ thể, ta có thể xây dựng một View hoặc truy vấn thẳng vào bảng Log như sau:
```sql
-- Truy xuất toàn bộ lịch sử trả nợ của Hợp Đồng số 1
SELECT 
    lg.MaLog,
    hd.MaHD,
    kh.TenKH,
    lg.NgayTra,
    lg.SoTienTra,
    lg.NguoiThuTien,
    lg.GhiChu
FROM LogGiaoDich lg
JOIN HopDong hd ON lg.MaHD = hd.MaHD
JOIN KhachHang kh ON hd.MaKH = kh.MaKH
WHERE hd.MaHD = 1
ORDER BY lg.NgayTra DESC;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/8e38894d-3ace-4356-bcb6-6ddc4507f0af" />

Kết quả của truy vấn trên sẽ hiển thị rõ ràng từng giao dịch khách hàng thực hiện (như việc trả 5 triệu ở Event 3 hoặc trả tiền lãi khi gia hạn).
