# Lộ trình học .NET để hiểu dự án này

> Mục tiêu: giúp người mới .NET hiểu nhanh cấu trúc, công nghệ và cách vận hành của hệ thống microservices trong repo này.

## 1. Làm quen tổng quan dự án (1–2 giờ)
**Tài liệu nên đọc trước:**
- `README.md`: mục tiêu dự án, kiến trúc tổng thể, các microservice và công nghệ chính.

**Kết quả mong đợi:**
- Hiểu hệ thống là gì, có những service nào, và cách chúng giao tiếp.

## 2. Nắm nền tảng .NET & C# cơ bản (1–2 tuần)
**Nội dung cốt lõi:**
- C# cơ bản: class, interface, async/await, DI, LINQ.
- ASP.NET Core căn bản: Controller, Routing, Middleware.

**Kết quả mong đợi:**
- Đọc được code trong các service ASP.NET Core.

## 3. Hiểu cấu trúc solution & các project (0.5–1 ngày)
**Bắt đầu từ:**
- `DotNetMicroservicesPoc.sln` để thấy danh sách project/service.

**Kết quả mong đợi:**
- Biết mỗi service nằm ở đâu, vai trò của `.Api`, `.Test`.

## 4. Học EF Core qua ProductService (2–4 ngày)
**Service nên bắt đầu:** `ProductService`
- Cấu hình EF Core trong `AddEFConfiguration`.
- `ProductDbContext` và mapping `ProductConfig`.
- `ProductRepository` thao tác CRUD cơ bản.

**Kết quả mong đợi:**
- Biết cách dự án cấu hình DbContext, chọn InMemory/SQL Server, và thao tác dữ liệu.

## 5. Hiểu giao tiếp giữa service (2–4 ngày)
**Trọng tâm:**
- REST sync: `PolicyService` gọi `PricingService`.
- RabbitMQ async: publish/subscribe events.
- Service discovery với Eureka.

**Kết quả mong đợi:**
- Hiểu luồng dữ liệu giữa các service trong runtime.

## 6. Học các storage khác (tùy chọn, 1–2 tuần)
**Theo service:**
- PostgreSQL + NHibernate (`PolicyService`).
- Marten (Postgres document DB) (`PricingService`, `PaymentService`).
- Elasticsearch (`PolicySearchService`, `DashboardService`).

## 7. Làm quen API Gateway & cache (0.5–1 ngày)
**AgentPortalApiGateway:**
- Ocelot làm API Gateway.
- Cache route bằng Ocelot CacheManager (in-memory).

## 8. Chạy hệ thống & thực hành (1–2 ngày)
- Dùng Docker scripts trong thư mục `scripts/`.
- Thử chạy riêng từng service bằng `dotnet run`.

---

## Gợi ý thứ tự học ngắn gọn
1. README.md (tổng quan)
2. .NET/C# cơ bản
3. ProductService + EF Core
4. Giao tiếp microservices (REST + RabbitMQ)
5. Storage khác (NHibernate/Marten/Elastic)
6. API Gateway + Cache

