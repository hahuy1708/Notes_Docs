# Stored Procedure

## sp_CreateOrder

- **Mục đích**: Xử lý việc tạo hóa đơn bán hàng kết hợp với cập nhật tồn kho và ghi nhận chi tiết đơn hàng.
  
**Các bước thực hiện:**

### 1. Kiểm tra đầu vào:

- Xác thực rằng các tham số bắt buộc (@CustomerID, @EmployeeID, @ProductList) không được để trống.
- Nếu thiếu thông tin, sử dụng lệnh THROW để trả về lỗi.

### 2. Xác thực sự tồn tại của khách hàng và nhân viên:

- Truy vấn bảng Customer và Employee để đảm bảo khởi tạo đơn hàng chỉ với những người đã tồn tại trong hệ thống.

### 3. Xử lý JSON:

- Sử dụng hàm OPENJSON để phân tích chuỗi JSON chứa danh sách sản phẩm.
- Các sản phẩm được tải vào một bảng tạm @OrderProducts với các cột: Product_Id, Quantity, và Price.
  
### 4. Kiểm tra tồn tại của sản phẩm và số lượng:

- So sánh các Product_Id nhận được với bảng Product. Nếu có sản phẩm không tồn tại, tổng hợp thông báo lỗi và ném ngoại lệ.
- Kiểm tra tồn kho của từng sản phẩm, nếu số lượng trong kho không đủ so với yêu cầu đặt hàng, cũng sẽ phát sinh lỗi thông báo cụ thể.

### 5. Tạo hóa đơn bán hàng:

- Chèn một bản ghi vào bảng Receipt với thông tin khách hàng, nhân viên, ngày tạo và tổng hóa đơn.  
- Lấy mã hóa đơn vừa tạo qua SCOPE_IDENTITY().

### 6. Ghi nhận chi tiết đơn hàng:

- Chèn các dòng chi tiết đơn hàng từ bảng tạm @OrderProducts vào bảng Details, liên kết với hóa đơn vừa tạo qua Receipt_Id.

### 7. Cập nhật tồn kho:

- Giảm số lượng tồn kho trong bảng Product tương ứng với số lượng của sản phẩm đặt hàng.

### 8. Giao dịch:

- Tất cả các thao tác trên được thực hiện bên trong một giao dịch (BEGIN TRANSACTION và COMMIT TRANSACTION) để đảm bảo tính toàn vẹn dữ liệu. Nếu có lỗi bất kỳ, giao dịch sẽ được ROLLBACK trong khối CATCH.

------------------------------------



