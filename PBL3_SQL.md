# Stored Procedure - Transaction

## sp_CreateOrder

- **Mục đích**: Xử lý việc tạo hóa đơn bán hàng kết hợp với cập nhật tồn kho và ghi nhận chi tiết đơn hàng.

**Các bước thực hiện:**

### 1. Kiểm tra và xử lý đầu vào:

- Nhận chuỗi JSON chứa danh sách sản phẩm đặt hàng, gồm các trường `productId`, `quantity`, `price`, và `customerId`.
- Dữ liệu JSON được phân tích qua `OPENJSON` và tải vào bảng tạm `@OrderProducts`.
- Kiểm tra xem bảng tạm có dữ liệu không (báo lỗi nếu danh sách trống).
- Xác thực rằng các tham số bắt buộc (`customerId`, `employeeId`) không được để trống.

### 2. Xác thực dữ liệu:

- Kiểm tra sự tồn tại của khách hàng (`Customer`) dựa vào `customerId`.
- Kiểm tra sự tồn tại của nhân viên (`Employee`) thực hiện thao tác bán hàng.
- Nếu bất kỳ điều kiện nào không hợp lệ, sử dụng `THROW` để ném lỗi.

### 3. Giao dịch và tạo hóa đơn:

- Bắt đầu giao dịch với `BEGIN TRANSACTION` để đảm bảo tất cả các thao tác được thực hiện nguyên tử.
- Tạo một bản ghi mới trong **Receipt** để lưu ngày tạo hóa đơn, nhân viên thực hiện, và khách hàng đặt hàng.
- Lấy mã hóa đơn mới tạo qua `SCOPE_IDENTITY()`.

### 4. Kiểm tra tính hợp lệ của sản phẩm:

- So sánh từng `ProductId` trong bảng tạm với bảng **Product**.
- Nếu có sản phẩm không tồn tại, tổng hợp thông báo lỗi và `ROLLBACK TRANSACTION`.
- Kiểm tra tồn kho của từng sản phẩm, nếu số lượng đặt vượt quá tồn kho, thông báo lỗi và `ROLLBACK TRANSACTION`.

### 5. Ghi nhận chi tiết đơn hàng:

- Chèn dữ liệu vào bảng **Details**, gồm thông tin sản phẩm, số lượng, giá bán và liên kết với hóa đơn (`Receipt_Id`).

### 6. Cập nhật tồn kho:

- Giảm số lượng tồn kho của sản phẩm trong bảng **Product** bằng số lượng đặt hàng.

### 7. Kết thúc giao dịch:

- Nếu tất cả các bước trên đều thành công, giao dịch được `COMMIT`.
- Nếu có lỗi trong quá trình thực hiện, giao dịch sẽ được `ROLLBACK`.

### 8. Trả thông tin kết quả:

- Truy vấn bảng **Receipt**, **Customer**, **Employee**, và **Details** để trả về thông tin tổng hợp của hóa đơn bán hàng.
- Truy vấn chi tiết từng sản phẩm đặt gồm số lượng, đơn giá, và tổng giá trị đơn hàng.
  
------------------------------------

## sp_ImportProducts


- **Mục đích**: Xử lý quá trình nhập hàng từ nhà cung cấp, bao gồm cập nhật tồn kho, ghi nhận phiếu nhập hàng (Goods Receipt) và lưu trữ chi tiết nhập hàng.

**Các bước thực hiện:**

### 1. Kiểm tra và xử lý đầu vào:

- Nhận chuỗi JSON chứa danh sách sản phẩm nhập hàng, gồm các trường `productId`, `quantity`, `price`, và `supplierId`.
- Dữ liệu JSON được phân tích qua `OPENJSON` và tải vào bảng tạm `@ProductDetails`.
- Kiểm tra xem bảng tạm có dữ liệu không (báo lỗi nếu danh sách trống).
- Đảm bảo rằng tất cả sản phẩm có cùng `supplierId` và giá trị này không được để trống.

### 2. Xác thực dữ liệu:

- Kiểm tra sự tồn tại của nhà cung cấp (`Supplier`) dựa vào `supplierId` lấy từ danh sách.
- Kiểm tra sự tồn tại của nhân viên (`Employee`) thực hiện thao tác nhập hàng.
- Nếu bất kỳ điều kiện nào không hợp lệ, sử dụng `THROW` để ném lỗi.

### 3. Giao dịch và ghi nhận phiếu nhập hàng:

- Bắt đầu giao dịch với `BEGIN TRANSACTION` để đảm bảo tất cả các thao tác được thực hiện nguyên tử.
- Tạo một bản ghi mới trong **Goods_Receipt** để lưu ngày nhập hàng và nhân viên nhập hàng.
- Lấy mã phiếu nhập hàng mới tạo qua `SCOPE_IDENTITY()`.

### 4. Kiểm tra tính hợp lệ của sản phẩm:

- So sánh từng `ProductId` trong bảng tạm với bảng **Product**.
- Nếu có sản phẩm không tồn tại, tổng hợp thông báo lỗi và `ROLLBACK TRANSACTION`.

### 5. Ghi nhận chi tiết nhập hàng:

- Chèn dữ liệu vào bảng **Details**, gồm thông tin sản phẩm, số lượng, giá nhập và liên kết với phiếu nhập hàng (`GoodsReceipt_Id`).

### 6. Cập nhật tồn kho:

- Tăng số lượng tồn kho của sản phẩm trong bảng **Product** bằng số lượng nhập từ danh sách.

### 7. Kết thúc giao dịch:

- Nếu tất cả các bước trên đều thành công, giao dịch được `COMMIT`.
- Nếu có lỗi trong quá trình thực hiện, giao dịch sẽ được `ROLLBACK`.

### 8. Trả thông tin kết quả:

- Truy vấn bảng **Goods_Receipt**, **Supplier**, **Employee**, và **Details** để trả về thông tin tổng hợp của phiếu nhập hàng.
- Truy vấn chi tiết từng sản phẩm nhập gồm số lượng, đơn giá, và tổng giá trị nhập.

## Đặc điểm kỹ thuật

- **Tính toàn vẹn dữ liệu**:
  - Sử dụng transaction để đảm bảo rằng dữ liệu được cập nhật hợp lệ và có thể quay lui nếu có lỗi.
  - Kiểm tra tính hợp lệ của dữ liệu đầu vào trước khi thực hiện cập nhật vào database.
  - Kiểm soát lỗi bằng cách sử dụng `THROW` và `TRY...CATCH`.

- **Tối ưu hóa hiệu suất**:
  - Sử dụng JSON để truyền danh sách sản phẩm, giúp đơn giản hóa quá trình xử lý dữ liệu.
  - Cập nhật tồn kho một cách hiệu quả bằng `INNER JOIN` giữa bảng **Product** và bảng tạm chứa sản phẩm nhập.
  - Sử dụng `SCOPE_IDENTITY()` để lấy ID mới tạo thay vì `@@IDENTITY`, tránh lỗi khi có triggers.

------------------------------------
# Cascade

- Khi một bản ghi trong bảng cha (bảng chứa khóa chính) bị xóa, các bản ghi liên quan trong bảng con (bảng chứa khóa ngoại) sẽ tự động bị xóa theo. Điều này giúp đảm bảo rằng sau khi xóa bản ghi cha, không còn các bản ghi "mồ côi" (orphan records) vi phạm tính toàn vẹn của dữ liệu.

1.  Bảng Accessories, Laptop, PC và Product:

```sql
ALTER TABLE [dbo].[Accessories] WITH NOCHECK ADD FOREIGN KEY([Product_Id])
REFERENCES [dbo].[Product] ([Product_Id])
ON DELETE CASCADE
ALTER TABLE [dbo].[Laptop] WITH NOCHECK ADD FOREIGN KEY([Product_Id])
REFERENCES [dbo].[Product] ([Product_Id])
ON DELETE CASCADE
GO
ALTER TABLE [dbo].[PC] WITH NOCHECK ADD FOREIGN KEY([Product_Id])
REFERENCES [dbo].[Product] ([Product_Id])
ON DELETE CASCADE
```
->  Khi một sản phẩm trong bảng Product bị xóa, các bản ghi tương ứng trong bảng Accessories,Laptop, PC (liên kết với sản phẩm đó) sẽ tự động bị xóa. Qua đó, đảm bảo không có phụ kiện liên kết tới một sản phẩm không tồn tại nữa.

2. Bảng Details liên kết với Goods_Receipt và Receipt:

```sql
ALTER TABLE [dbo].[Accessories] WITH NOCHECK ADD FOREIGN KEY([Product_Id])
REFERENCES [dbo].[Product] ([Product_Id])
ON DELETE CASCADE
ALTER TABLE [dbo].[Laptop] WITH NOCHECK ADD FOREIGN KEY([Product_Id])
REFERENCES [dbo].[Product] ([Product_Id])
ON DELETE CASCADE
GO
ALTER TABLE [dbo].[PC] WITH NOCHECK ADD FOREIGN KEY([Product_Id])
REFERENCES [dbo].[Product] ([Product_Id])
ON DELETE CASCADE
```
-> Nếu một phiếu nhập hàng (Goods_Receipt) bị xóa, hoặc một hóa đơn (Receipt) bị xóa, thì các chi tiết liên quan trong bảng Details sẽ bị xóa tự động. Điều này giúp tránh trường hợp tồn tại các dòng chi tiết mà không còn phiếu nhập hay hóa đơn làm chủ.


### WITH NOCHECK - WITH CHECK

- **WITH NOCHECK** Được sử dụng khi thêm ràng buộc (`FOREIGN KEY`, `CHECK`), giúp **bỏ qua việc kiểm tra dữ liệu hiện có** nhưng vẫn áp dụng ràng buộc cho dữ liệu mới.  
-> Không kiểm tra các bản ghi hiện có để đảm bảo chúng tuân thủ ràng buộc mới.
 
- **WITH CHECK**: Kiểm tra dữ liệu hiện có trước khi áp dụng ràng buộc.Đảm bảo tất cả các Account_Id của nhân viên đều hợp lệ.


# Mối quan hệ và ràng buộc khóa ngoại

1. Employee - Account (1-1): 
Mỗi nhân viên (Employee) có một tài khoản đăng nhập (Account).
2. Goods_Receipt - Employee (1-N): 
Một nhân viên (Employee) có thể lập nhiều phiếu nhập hàng (Goods_Receipt).
3. laptop - Product, PC - Product (1-1):
Một sản phẩm (Product) có thể là Laptop hoặc PC, nhưng mỗi sản phẩm chỉ thuộc một loại.
4. Product - Supplier (N-1): 
Một nhà cung cấp (Supplier) có thể cung cấp nhiều sản phẩm (Product)
5. Receipt - Customer, Receipt - Employee (1-N):
Một khách hàng (Customer) có thể tạo nhiều hóa đơn (Receipt).
Một nhân viên (Employee) có thể phụ trách nhiều hóa đơn.


```sql
ALTER TABLE [dbo].[Details] WITH NOCHECK ADD CHECK (([productPrice]>=(0)))
GO
ALTER TABLE [dbo].[Details] WITH NOCHECK ADD CHECK (([quantity]>(0)))
GO
```

- Đảm bảo rằng giá sản phẩm (productPrice) không thể nhỏ hơn 0 (giá không hợp lệ).
- Đảm bảo rằng số lượng (quantity) phải lớn hơn 0 (không thể mua/bán số lượng âm).
- Giúp duy trì tính hợp lệ của dữ liệu.





