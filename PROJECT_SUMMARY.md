# Tổng quan dự án dotnetcore-microservices-poc

## 1. Mục tiêu & bối cảnh
Dự án là một hệ thống bán bảo hiểm mẫu theo kiến trúc microservices. Nhiệm vụ chính gồm:
- Đăng nhập/ủy quyền cho nhân viên bán hàng.
- Chọn sản phẩm bảo hiểm, tính giá, tạo offer và chuyển đổi thành policy.
- Tra cứu policy/offer, giao tiếp realtime (chat), và dashboard thống kê doanh số.

## 2. Kiến trúc tổng thể
- Kiến trúc **microservices** với **API Gateway** (AgentPortalApiGateway).
- Giao tiếp:
  - Đồng bộ: REST (PolicyService gọi PricingService).
  - Bất đồng bộ: RabbitMQ (publish/subscribe events).
- Service discovery: **Eureka**.
- Search/analytics: **Elasticsearch**.
- Cache ở gateway: **Ocelot CacheManager** với in-memory dictionary handle.

## 3. Tech stack chính
- **.NET 8**
- **Entity Framework Core**, **Marten**, **NHibernate**, **Dapper**
- **MediatR**, **CQRS** (theo mô tả kiến trúc)
- **Eureka**, **Ocelot**
- **JWT**, **RestEase**, **RawRabbit**
- **Polly**, **NEST/Elasticsearch client**
- **DynamicExpresso**, **SignalR**

## 4. Cấu trúc solution & các service
Các project chính trong `DotNetMicroservicesPoc.sln`:
- **AgentPortalApiGateway**: API Gateway.
- **AuthService**: xác thực/ủy quyền (JWT).
- **ChatService**: chat realtime.
- **PaymentService**: quản lý payment/account, background jobs.
- **PolicyService**: quản lý offer & policy, CQRS.
- **PolicySearchService**: tìm kiếm policy.
- **PricingService**: tính giá.
- **ProductService**: catalog sản phẩm.
- **DashboardService**: thống kê doanh số.

Mỗi microservice có thể có:
- Project `.Api` (commands/events/queries/operations).
- Project `.Test` (unit/integration test).

## 5. CSDL theo từng service
### 5.1 PostgreSQL
- **PaymentService**: Postgres (db: `lab_netmicro_payments`), Hangfire jobs dùng `lab_netmicro_jobs`.
- **PolicyService**: Postgres (db: `lab_netmicro_policy`).
- **PricingService**: Postgres (db: `lab_netmicro_pricing`).

### 5.2 Elasticsearch
- **PolicySearchService**: Elasticsearch (search index cho policy).
- **DashboardService**: Elasticsearch (aggregation cho thống kê).

### 5.3 InMemory
- **ProductService**: cấu hình dùng InMemory database (EF Core) trong appsettings.

### 5.4 Không thấy cấu hình DB
- **AuthService**
- **ChatService**
- **AgentPortalApiGateway**

> Lưu ý: Không thấy MySQL hay MongoDB trong các cấu hình hiện tại.

## 6. Cache/memory cache
- **AgentPortalApiGateway** dùng **Ocelot CacheManager** với cấu hình cache in-memory (dictionary handle) để cache route (TTL 15s trong `ocelot.json`). Không thấy Redis hoặc distributed cache khác trong cấu hình hiện tại.

## 7. Các framework data-access theo service
- **PaymentService**: Dapper, Marten, EF Core (SqlServer).
- **PolicyService**: NHibernate.
- **PricingService**: Marten.
- **ProductService**: EF Core.
- **PolicySearchService**: Elastic.Clients.Elasticsearch.
- **DashboardService**: Elastic.Clients.Elasticsearch.
- **AuthService/ChatService/AgentPortalApiGateway**: không có framework DB rõ ràng trong `.csproj`.

## 8. Tài liệu/đường dẫn tham khảo chính
- README.md mô tả kiến trúc, tech stack, và hướng dẫn chạy Docker/manual.
- `postgres/createdatabases.sql` khai báo các database Postgres dùng trong hệ thống.
