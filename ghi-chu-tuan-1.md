# GHI CHÚ BÀI TẬP TUẦN 1 - ABP FRAMEWORK (BOOK STORE)

---

## 1. Vai trò của từng project trong Layered Solution

Dự án được xây dựng theo kiến trúc **Domain-Driven Design (DDD)** của ABP Framework, chia thành các layer chính:

* **Domain.Shared**: 
  Chứa các enum (ví dụ: `BookType`), hằng số (constants), struct hoặc class dùng chung cho toàn bộ hệ thống (cả frontend lẫn backend) và chứa tài liệu Localization (đa ngôn ngữ).
* **Domain**: 
  Chứa trái tim của ứng dụng bao gồm các Entity (`Book` kế thừa `AuditedAggregateRoot<Guid>`), Value Objects, Domain Services, Repository Interfaces và Business Rules (quy tắc nghiệp vụ chính).
* **Application.Contracts**: 
  Khai báo các Interface (`IBookAppService`), Data Transfer Objects (`BookDto`, `CreateUpdateBookDto`), Permissions và hằng số định nghĩa giao tiếp API.
* **Application**: 
  Cài đặt (Implement) logic nghiệp vụ từ `Application.Contracts` (`BookAppService` kế thừa `CrudAppService`). Chuyển đổi dữ liệu giữa Entity và DTO (Object Mapping via AutoMapper).
* **EntityFrameworkCore**: 
  Cấu hình cơ sở dữ liệu EF Core, `DbContext`, tích hợp `AbpEntityFrameworkCoreMySQLModule`, định nghĩa quan hệ bảng và quản lý các file Migrations cho MySQL.
* **HttpApi**: 
  Tự động ánh xạ (Map) các Application Services thành các RESTful API Controllers tiêu chuẩn.
* **HttpApi.Host**: 
  Project khởi chạy (Startup) chính của Backend API, cấu hình Middleware, Dependency Injection, Authentication/Authorization, CORS và hiển thị Swagger UI.
* **DbMigrator**: 
  Console application độc lập chịu trách nhiệm tự động cập nhật Database Migrations và Seed dữ liệu ban đầu (`BookStoreDataSeederContributor`) cho hệ thống.

---

## 2. Mô tả chi tiết Luồng xử lý CRUD Book qua Swagger API

### A. Luồng Tạo mới sách (Create - `POST /api/app/book`)
1. **Client (Swagger UI)**: Người dùng gửi HTTP Request `POST` kèm Body JSON chứa `CreateUpdateBookDto` (Name, Type, PublishDate, Price).
2. **HttpApi Host Layer**: Tiếp nhận Request, Router điều hướng tới Dynamic Controller đại diện cho `IBookAppService`.
3. **Application Layer (`BookAppService`)**:
   - Tự động kiểm tra Validation của `CreateUpdateBookDto`.
   - Gọi AutoMapper để map từ `CreateUpdateBookDto` sang Entity `Book`.
4. **Domain & EF Core Layer**:
   - `BookAppService` gọi `IRepository<Book, Guid>` thực hiện lệnh `InsertAsync(book)`.
   - EF Core biên dịch Entity thành câu lệnh SQL `INSERT INTO AppBooks (...) VALUES (...)` và thực thi trên MySQL.
5. **Response**: EF Core lưu thành công, `BookAppService` map Entity `Book` vừa tạo thành `BookDto` và trả về JSON cho Swagger UI với HTTP Status Code `201 Created`.

---

### B. Luồng Lấy danh sách sách có phân trang (GetList - `GET /api/app/book`)
1. **Client (Swagger UI)**: Gửi Request `GET` kèm các tham số phân trang (`SkipCount`, `MaxResultCount`, `Sorting`).
2. **Application Layer (`BookAppService`)**:
   - `CrudAppService` tiếp nhận tham số `PagedAndSortedResultRequestDto`.
   - Tạo câu vấn truy vấn LINQ trỏ tới `IRepository<Book, Guid>`.
3. **EF Core Layer**:
   - Chuyển LINQ thành SQL `SELECT * FROM AppBooks ORDER BY ... LIMIT ... OFFSET ...` và lấy tổng số bản ghi (`COUNT`).
4. **Response**: Map danh sách Entity `Book` thành `List<BookDto>`, đóng gói vào object `PagedResultDto<BookDto>` (chứa `TotalCount` và danh sách `Items`) trả về JSON với HTTP `200 OK`.

---

### C. Luồng Xem chi tiết sách theo ID (Get - `GET /api/app/book/{id}`)
1. **Client (Swagger UI)**: Gửi Request `GET` kèm chuỗi `Guid id` trên URL.
2. **Application Layer**: `BookAppService` gọi `GetAsync(id)`.
3. **EF Core Layer**: Chạy câu lệnh SQL `SELECT * FROM AppBooks WHERE Id = @id`.
4. **Response**: Nếu tìm thấy, map Entity `Book` sang `BookDto` trả về HTTP `200 OK`. Nếu không thấy, ABP tự động bắn ra `EntityNotFoundException` và trả về HTTP `404 Not Found`.

---

### D. Luồng Cập nhật sách (Update - `PUT /api/app/book/{id}`)
1. **Client (Swagger UI)**: Gửi Request `PUT` kèm `id` trên URL và Body JSON `CreateUpdateBookDto` mới.
2. **Application Layer**:
   - `BookAppService` lấy Entity `Book` hiện tại từ Database qua `GetAsync(id)`.
   - Áp dụng AutoMapper để cập nhật các thuộc tính từ DTO vào Entity `Book`.
3. **EF Core Layer**: Chạy câu lệnh SQL `UPDATE AppBooks SET Name = ..., Price = ... WHERE Id = @id`.
4. **Response**: Trả về `BookDto` đã cập nhật thông tin mới với HTTP `200 OK`.

---

### E. Luồng Xóa sách (Delete - `DELETE /api/app/book/{id}`)
1. **Client (Swagger UI)**: Gửi Request `DELETE` kèm `id` cần xóa.
2. **Application Layer & EF Core**:
   - `BookAppService` gọi `DeleteAsync(id)`.
   - Do Entity `Book` kế thừa `AuditedAggregateRoot` (hoặc `FullAuditedAggregateRoot`), ABP sẽ thực hiện **Soft Delete** (cập nhật trường `IsDeleted = true`) hoặc xóa cứng tùy theo cấu hình.
3. **Response**: Trả về HTTP `204 No Content` báo hiệu xóa thành công.

---

## 3. Các bước Khởi chạy & Kiểm thử ứng dụng

1. **Khởi tạo Database MySQL**:
   ```cmd
   dotnet ef database update -p src/Acme.BookStore.EntityFrameworkCore -s src/Acme.BookStore.DbMigrator